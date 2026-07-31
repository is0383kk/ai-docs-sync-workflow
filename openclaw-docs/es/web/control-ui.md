---
read_when:
    - Quiere operar el Gateway desde un navegador
    - Quieres acceder a la Tailnet sin túneles SSH
sidebarTitle: Control UI
summary: Interfaz de control basada en navegador para el Gateway (chat, actividad, nodos, configuración)
title: Interfaz de control
x-i18n:
    generated_at: "2026-07-26T05:34:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 069bad7f3c8fce46759893e16d2dac86047c0929d6d866d25ce3b080204c1180
    source_path: web/control-ui.md
    workflow: 16
---

La interfaz de control es una pequeña aplicación de página única de **Vite + Lit** servida por el Gateway:

- valor predeterminado: `http://<host>:18789/`
- prefijo opcional: establezca `gateway.controlUi.basePath` (p. ej., `/openclaw`)

Se comunica **directamente con el WebSocket del Gateway** en el mismo puerto.

Mientras se observa una sesión en ejecución, el Gateway puede usar el modelo auxiliar de ese agente para generar un resumen de estado compacto. El chat lo muestra como una insignia de estado de una línea que se expande en una tarjeta con la evaluación, el progreso del plan, los pull requests y el tiempo transcurrido. La tarjeta puede expandirse una vez cuando una ejecución se bloquea o necesita información; el chat lateral `/btw` tiene prioridad sobre la tarjeta expandida.

La tarjeta expandida también acepta preguntas breves sobre la ejecución. Las respuestas usan únicamente el resumen actual del observador y notas acotadas y depuradas, permanecen en el navegador durante esa sesión y nunca entran en la ejecución principal del agente ni la interrumpen. Si las observaciones no contienen la respuesta, el observador indica que no puede saberla.

Cuando llega el primer resumen, este pasa a controlar el subtítulo de esa ejecución en la barra lateral en lugar de la actividad en vivo heurística. Un resumen final de ejecución completada o fallida permanece visible mientras la sesión no se haya leído; después, la fila vuelve a mostrar su subtítulo de trabajo normal.

La observación de sesiones está habilitada de forma predeterminada. En **Configuración > Apariencia > Barra lateral**, se puede desactivar para todo el Gateway, consultar el modelo pequeño resuelto y su procedencia, elegir el enrutamiento automático, deshabilitar las tareas auxiliares o seleccionar un `agents.defaults.utilityModel` explícito. Los controles de configuración equivalentes son `gateway.controlUi.sessionObserver: false` y `agents.defaults.utilityModel: ""`.

## Apertura rápida (local)

Si el Gateway se ejecuta en el mismo equipo, abra [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (o [http://localhost:18789/](http://localhost:18789/)).

Si la página no se carga, inicie primero el Gateway: `openclaw gateway`.

<Note>
En vinculaciones de LAN nativas de Windows, el Firewall de Windows o la directiva de grupo administrada por la organización pueden bloquear la URL de LAN anunciada incluso cuando `127.0.0.1` funciona en el host del Gateway. Ejecute `openclaw gateway status --deep` en el host de Windows; informa de puertos probablemente bloqueados, discrepancias de perfiles y reglas de firewall locales que la directiva podría ignorar.
</Note>

La autenticación se proporciona durante el protocolo de enlace de WebSocket mediante:

- `connect.params.auth.token`
- `connect.params.auth.password`
- encabezados de identidad de Tailscale Serve cuando `gateway.auth.allowTailscale: true`
- encabezados de identidad de proxy de confianza cuando `gateway.auth.mode: "trusted-proxy"`

La autenticación del Gateway se ejecuta antes del emparejamiento del dispositivo. Una conexión directa de bucle invertido no omite la autenticación mediante token o contraseña. El panel de configuración conserva un token para la sesión actual de la pestaña del navegador y la URL del Gateway seleccionada; las contraseñas no se conservan. Tras el emparejamiento, el navegador puede usar su token almacenado por dispositivo en conexiones posteriores.

La incorporación suele configurar un token del Gateway para la autenticación mediante secreto compartido. Si el Gateway se inicia en modo de token sin un token configurado, genera en su lugar un token efímero de tiempo de ejecución para ese proceso. El token de tiempo de ejecución no se escribe en la configuración, por lo que `openclaw config get gateway.auth.token` no puede recuperarlo y se rechaza un navegador de bucle invertido que no tenga ese token. Ejecute `openclaw doctor --generate-gateway-token`, reinicie el Gateway y pegue después el token configurado en la configuración de la interfaz de control. También se puede usar la autenticación mediante contraseña cuando `gateway.auth.mode` es `"password"`.

## Emparejamiento de dispositivos (primera conexión)

Después de que la autenticación del Gateway se complete correctamente, la conexión desde un navegador o dispositivo nuevo suele requerir una **aprobación de emparejamiento única**, mostrada como `disconnected (1008): pairing required`.

<Warning>
Al actualizar directamente desde una versión que utilizaba la opción de emergencia retirada
`gateway.controlUi.dangerouslyDisableDeviceAuth=true`,
OpenClaw mantiene disponible el acceso a la interfaz de control autenticado mediante token/contraseña o proxy de confianza
únicamente para corregir el emparejamiento. Si el navegador usa HTTP sin cifrar y no puede crear una identidad de dispositivo,
vuelva a abrirlo primero mediante HTTPS o localhost. Después, haga clic en **Proteger este navegador** en
el banner de advertencia. El Gateway vuelve a aplicar normalmente la autenticación de dispositivos solo
después de que se empareje explícitamente un navegador firmado; nunca crea ni aprueba una
identidad para un navegador sin dispositivo. La transición no está disponible cuando
ya hay otro dispositivo de operador emparejado. Tanto el inicio del Gateway como
`openclaw doctor --fix` informan explícitamente de esta migración en lugar de
descartar silenciosamente la clave anterior.
</Warning>

<Steps>
  <Step title="Enumerar solicitudes pendientes">
    ```bash
    openclaw devices list
    ```
  </Step>
  <Step title="Aprobar mediante el ID de solicitud">
    ```bash
    openclaw devices approve <requestId>
    ```
  </Step>
</Steps>

Si el navegador vuelve a intentar el emparejamiento con datos de autenticación modificados (rol/ámbitos/clave pública), la solicitud pendiente anterior queda reemplazada y se crea un nuevo `requestId`; vuelva a ejecutar `openclaw devices list` antes de aprobarla.

El cambio de un navegador remoto ya emparejado desde acceso de lectura a acceso de escritura/administración se trata como una ampliación de aprobación, no como una reconexión silenciosa: OpenClaw mantiene activa la aprobación anterior, bloquea la reconexión con permisos más amplios y solicita que se apruebe explícitamente el nuevo conjunto de ámbitos. Una conexión de bucle invertido directo válida de la interfaz de control puede aprobar silenciosamente la ampliación después de autenticarse.

Una vez aprobado, el dispositivo queda recordado y no requerirá una nueva aprobación, salvo que se revoque mediante `openclaw devices revoke --device <id> --role <role>`. Consulte la [CLI de dispositivos](/es/cli/devices) para obtener información sobre la rotación y revocación de tokens y el flujo de aprobación de la primera ejecución de Paperclip / `openclaw_gateway`.

<Note>
- Las conexiones locales directas de la interfaz de control desde un par TCP de bucle invertido (`127.0.0.1` o `::1`, normalmente accesible como `localhost`) sin encabezados reenviados o de proxy pueden aprobar automáticamente el emparejamiento del dispositivo solo después de que la autenticación del Gateway se complete correctamente y el navegador presente una identidad de dispositivo. En el modo de token/contraseña, la primera conexión sigue necesitando el secreto compartido configurado; esta aprobación automática no omite el token.
- El bucle invertido directo no necesita ningún secreto compartido solo cuando `gateway.auth.mode: "none"` está configurado explícitamente. Esto deshabilita la autenticación del Gateway y no es la configuración recomendada para la interfaz de control. Los modos Tailscale Serve y proxy de confianza pueden evitar que se pegue un secreto compartido únicamente cuando sus respectivas comprobaciones de identidad se completan correctamente.
- Tailscale Serve puede omitir el proceso de ida y vuelta del emparejamiento para las sesiones de operador de la interfaz de control cuando `gateway.auth.allowTailscale: true`, se verifica la identidad de Tailscale y el navegador presenta su identidad de dispositivo. Los navegadores sin dispositivo y las conexiones con rol de Node siguen las comprobaciones normales de dispositivos.
- Las vinculaciones directas de Tailnet y las conexiones de navegador mediante LAN siguen requiriendo aprobación explícita. Los perfiles de navegador sin identidad de dispositivo no pueden usar la aprobación automática mediante bucle invertido.
- Cada perfil de navegador genera un ID de dispositivo único, por lo que cambiar de navegador o borrar sus datos requiere volver a realizar el emparejamiento.

</Note>

## Emparejar un dispositivo móvil

Un administrador ya emparejado puede crear el código QR de conexión para iOS/Android sin abrir una terminal:

<Steps>
  <Step title="Abrir el emparejamiento móvil">
    Seleccione **Dispositivos** y haga clic en **Emparejar dispositivo móvil** en la tarjeta **Dispositivos**.
  </Step>
  <Step title="Conectar el teléfono">
    En la aplicación móvil de OpenClaw, abra **Configuración** → **Gateway** y escanee el código QR. También se puede copiar y pegar el código de configuración.
  </Step>
  <Step title="Confirmar la conexión">
    La aplicación oficial para iOS/Android se conecta automáticamente. Si **Aprobación pendiente** muestra una solicitud, revise su rol y sus ámbitos antes de aprobarla.
  </Step>
</Steps>

La creación de un código de configuración requiere `operator.admin`; el botón está deshabilitado para las sesiones que no lo tengan. Un código de configuración contiene una credencial de arranque de corta duración, por lo que el código QR y el código copiado deben tratarse como una contraseña mientras sean válidos. Para el emparejamiento remoto, el Gateway debe resolverse como `wss://` (por ejemplo, mediante Tailscale Serve/Funnel); el valor `ws://` sin cifrar está limitado al bucle invertido y a direcciones LAN privadas. Consulte [Emparejamiento](/es/channels/pairing#pair-from-the-control-ui-recommended) para obtener todos los detalles de seguridad y alternativas.

## Identidad personal (local del navegador)

La interfaz de control admite una identidad personal por navegador (nombre para mostrar y avatar) asociada a los mensajes salientes para atribuir su autoría en sesiones compartidas. Se guarda en el almacenamiento del navegador, limitada al perfil de navegador actual, y no se sincroniza con otros dispositivos ni se conserva en el servidor más allá de los metadatos normales de autoría de la transcripción de los mensajes enviados. Al borrar los datos del sitio o cambiar de navegador, se restablece como vacía.

La sustitución del avatar del asistente sigue el mismo patrón local del navegador: las sustituciones cargadas se superponen localmente sobre la identidad resuelta por el Gateway y nunca realizan un recorrido de ida y vuelta a través de `config.patch`. El campo de configuración compartido `ui.assistant.avatar` sigue disponible para clientes sin interfaz de usuario que escriban el campo directamente.

## Endpoint de configuración de tiempo de ejecución

La interfaz de control obtiene su configuración de tiempo de ejecución desde `/control-ui-config.json`, resuelto con respecto a la ruta base de la interfaz de control del Gateway (por ejemplo, `/__openclaw__/control-ui-config.json` bajo la ruta base `/__openclaw__/`). Ese endpoint está protegido por la misma autenticación del Gateway que el resto de la superficie HTTP: los navegadores no autenticados no pueden acceder a él y, para obtenerlo correctamente, se requiere un token o contraseña válidos del Gateway, una identidad de Tailscale Serve o una identidad de proxy de confianza.

## Estado del host del Gateway

Abra **Configuración → General** para ver la tarjeta **Host del Gateway**, que muestra la máquina del Gateway, la dirección LAN, el sistema operativo, el entorno de ejecución, el tiempo de actividad, la carga de CPU, la memoria y el espacio en disco del volumen de estado. Mientras está visible, la tarjeta se actualiza cada 10 segundos mediante el RPC `system.info` del Gateway, que requiere el ámbito `operator.read`. Los Gateways antiguos y las conexiones sin ese ámbito omiten la tarjeta.

## Compatibilidad de idiomas

La interfaz de control se localiza durante la primera carga según la configuración regional del navegador. Para cambiarla posteriormente, abra **Configuración -> General -> Idioma** (el selector se encuentra en la página General, no en Apariencia).

- Configuraciones regionales compatibles: `en`, `ar`, `de`, `es`, `fa`, `fr`, `hi`, `id`, `it`, `ja-JP`, `ko`, `nl`, `pl`, `pt-BR`, `ru`, `th`, `tr`, `uk`, `vi`, `zh-CN`, `zh-TW`
- Las traducciones a idiomas distintos del inglés se cargan de forma diferida en el navegador.
- La configuración regional seleccionada se guarda en el almacenamiento del navegador y se reutiliza en visitas posteriores.
- Las claves de traducción que falten recurren al inglés.

Las traducciones de la documentación se generan para el mismo conjunto de configuraciones regionales distintas del inglés, pero el selector de idiomas integrado de Mintlify en el sitio de documentación solo muestra los códigos de configuración regional que Mintlify acepta. La documentación en tailandés (`th`) y persa (`fa`) también se genera en el repositorio de publicación; es posible que no aparezca en ese selector hasta que Mintlify admita esos códigos.

## Temas de apariencia

El panel Apariencia incluye los temas integrados Claw, Knot y Dash (Claw es el predeterminado), además de un espacio de importación de tweakcn local del navegador. Para importar un tema, abra el [editor de tweakcn](https://tweakcn.com/editor/theme), elija o cree un tema, haga clic en **Share** y pegue el enlace copiado en Apariencia. El importador también acepta URL de registro `https://tweakcn.com/r/themes/<id>`, URL del editor como `https://tweakcn.com/editor/theme?theme=amethyst-haze`, rutas relativas `/themes/<id>`, ID de tema sin procesar y nombres de temas predeterminados como `amethyst-haze`.

Los temas importados solo se almacenan en el perfil de navegador actual; no se escriben en la configuración del Gateway ni se sincronizan entre dispositivos. Al reemplazar el tema importado se actualiza el único espacio local; al borrarlo, se vuelve a Claw si el tema importado estaba activo.

Apariencia también incluye una opción de tamaño del texto. Se aplica al texto del chat, al texto del compositor, a las tarjetas de herramientas y a las barras laterales del chat, y mantiene las entradas de texto en un mínimo de 16px para que Safari móvil no amplíe automáticamente la vista al enfocarlas.

Las preferencias de tema, modo del tema, tamaño del texto, idioma y visualización del chat se sincronizan mediante la configuración del Gateway (`ui.prefs`), por lo que se mantienen en todos los dispositivos y los agentes pueden modificarlas a través de la puerta de aprobación; los clientes conectados aplican los cambios en tiempo real mediante el aviso `config.changed` del Gateway. Cada navegador conserva una copia local para iniciarse al instante; los clientes que no pueden escribir en la configuración (alcance de visualización, sin conexión) mantienen los cambios únicamente en el dispositivo. Consulte la [referencia de configuración](/es/gateway/configuration-reference#ui).

## Mantenimiento del sistema OpenClaw

Abra **Settings → Ask OpenClaw** para comunicarse con el agente de configuración y reparación del sistema. Fuera de la incorporación, esta página puede mostrar como máximo un indicador de evento descartable por visita. Permanece en silencio durante el tráfico rutinario del Gateway y solo reacciona a instantáneas de estado que informan de un recargador de configuración deshabilitado, la desconexión o degradación de un canal configurado, una comprobación de canal fallida o credenciales de canal no disponibles. Un evento más reciente sustituye el indicador pendiente solo cuando es más grave; descartar o utilizar el indicador silencia los avisos de eventos durante esa visita. Al hacer clic en el indicador, su pregunta de diagnóstico se envía como un mensaje `openclaw.chat` real, de modo que la transcripción registra la solicitud y OpenClaw realiza el diagnóstico. Estos indicadores de eventos nunca se muestran durante la incorporación.

## Gestionar plugins

Abra **Plugins** en la barra lateral o use `/settings/plugins` en relación con la
ruta base configurada de la interfaz de control para explorar y gestionar plugins sin salir
de la interfaz de control. Por ejemplo, una ruta base `/openclaw` utiliza
`/openclaw/settings/plugins`. La página siempre está disponible, incluso cuando todos los
plugins opcionales están deshabilitados.

Plugins es un centro con cuatro pestañas: **Instalados** y **Descubrir** gestionan el código de los plugins
en `/settings/plugins`, **Skills** aloja el gestor de Skills por agente en
`/skills` y **Taller** aloja la revisión de propuestas del Taller de Skills en
`/skills/workshop`. Cada pestaña conserva su propia URL y la barra lateral muestra la
única entrada Plugins para todas ellas.

La pestaña **Instalados** muestra el inventario local completo agrupado por categoría, con
recuentos generales. Cada fila abre una vista detallada; su menú de desbordamiento (`…`)
habilita o deshabilita el plugin y ofrece **Eliminar** para los plugins instalados externamente.
También enumera los [servidores MCP](/es/cli/mcp) configurados y permite añadirlos, deshabilitarlos
y eliminarlos en línea. Los mismos controles de servidor se encuentran en **Settings → MCP**.
La pestaña **Descubrir** es la tienda: plugins destacados incluidos con OpenClaw,
plugins externos oficiales y conectores MCP de un solo clic para servicios populares.
Al escribir en el cuadro de búsqueda, se consulta
[ClawHub](https://clawhub.ai/plugins) en línea y se añade una sección **De ClawHub**
con recuentos de descargas e insignias de verificación de la fuente. Los enlaces profundos pueden
dirigirse directamente a la tienda con `/settings/plugins?tab=discover`.

La pestaña **Skills** conserva el informe de estado de las Skills, los controles para habilitarlas o deshabilitarlas, la introducción de claves
de API y la búsqueda en línea de Skills de ClawHub, limitados al agente seleccionado. La
pestaña **Taller** contiene el tablero del Taller de Skills y el flujo de revisión de Hoy para
las [propuestas de Skills](/es/tools/skill-workshop). **Buscar ideas de Skills** revisa una ventana limitada
de sesiones sustanciales, desde las más recientes hasta las más antiguas, y deja los resultados como
propuestas pendientes. El panel muestra la cobertura acumulada; **Analizar trabajos anteriores**
continúa desde el cursor persistente y, una vez agotado el historial anterior, se convierte en **Analizar trabajos nuevos**.
La revisión manual del historial funciona mientras el autoaprendizaje autónomo
está deshabilitado y utiliza el modelo configurado del agente seleccionado.

Los plugins incluidos ya están presentes en el Gateway y muestran **Habilitar** o
**Deshabilitar** en lugar de **Instalar**. Por ejemplo, Workboard está incluido con
OpenClaw, pero está deshabilitado de forma predeterminada, por lo que su acción es **Habilitar**. Los plugins
integrados no pueden eliminarse, solo deshabilitarse.

La lectura del catálogo y la búsqueda en ClawHub requieren `operator.read`. Instalar,
habilitar, deshabilitar o eliminar un plugin y cambiar los servidores MCP requieren
`operator.admin`; estas acciones permanecen deshabilitadas para los operadores de solo lectura.

Las instalaciones desde ClawHub se ejecutan mediante el Gateway y mantienen las mismas comprobaciones de confianza, integridad
y políticas de instalación de plugins que otras instalaciones mediadas por el Gateway. Instalar
o eliminar código de plugins requiere reiniciar el Gateway. Habilitar o deshabilitar un
plugin instalado puede aplicarse sin reiniciar cuando el plugin y el entorno de ejecución actual
del Gateway lo admiten; de lo contrario, la interfaz informa de que es necesario
reiniciar. Los conectores MCP respaldados por OAuth necesitan ejecutar una vez
`openclaw mcp login <name>` desde la CLI después de añadirlos.

La página se centra intencionadamente en el inventario, el descubrimiento, la instalación, la habilitación
y la eliminación. Use [`openclaw plugins`](/es/cli/plugins) para fuentes arbitrarias de npm, git o
rutas locales, actualizaciones y configuración avanzada de plugins.

## Aplicaciones y extensiones

Abra **Aplicaciones** desde el menú **Más** de la barra lateral, la paleta de comandos o el
menú del agente de la barra lateral (**Obtener las aplicaciones**), o use `/apps` en relación con la
ruta base configurada de la interfaz de control. La página reúne enlaces de instalación para todas las
superficies complementarias de OpenClaw: las aplicaciones de [iOS](/es/platforms/ios) y
[Android](/es/platforms/android), los complementos de Apple Watch y Wear OS
incluidos con ellas, las aplicaciones de escritorio para [macOS](/es/platforms/macos), [Windows](/es/platforms/windows)
y [Linux](/es/platforms/linux), la
[extensión de Chrome](/es/tools/chrome-extension), el centro Plugins integrado en la aplicación con
[ClawHub](https://clawhub.ai), y la comunidad de Discord y la documentación.

## Navegación de la barra lateral

La barra lateral organiza todo en torno al agente. La fila de identidad de la parte superior corresponde al agente activo; debajo, la sección **Páginas** comienza con **Inicio** —la sesión principal continua del agente, con una insignia que indica su estado no leído o en ejecución—, seguida de los destinos fijados (**Automatizaciones** y **Plugins** de forma predeterminada). El control de personalización del encabezado Páginas abre un menú con todos los demás destinos, incluidos **Uso** y las pestañas proporcionadas por plugins, además de **Editar elementos fijados**; al hacer clic con el botón derecho en el área de navegación se abre directamente el editor de elementos fijados. La lista de sesiones inferior se divide en zonas: **Hilos** para las sesiones de chat del agente (la sesión principal permanece detrás de Inicio; las sesiones que esta inicia aparecen aquí como hilos de nivel superior y los hilos con nombre se muestran sin prefijo de tipo), **Grupos** para las conversaciones de grupos y salas, y **Programación** para las sesiones vinculadas a un árbol de trabajo gestionado o un Node de ejecución (las filas muestran una línea `repo ⎇ branch` además del host del Node), las sesiones de arnés respaldadas por ACP y los catálogos de las CLI de Codex/Claude. Programación comienza contraída en la primera ejecución y recuerda la elección; su encabezado contraído conserva el recuento real y muestra un indicador de ejecución mientras trabajan las sesiones que contiene. Los grupos personalizados (la sesión `category`) y las filas **Fijadas** se sitúan por encima de Hilos, y asignar una sesión a un grupo personalizado siempre prevalece sobre la clasificación automática por zonas. El encabezado Hilos contiene el control de ordenación (Creación o Última actualización, Agrupar por y un filtro persistente de **Estado** para Activas, Archivadas o Todas) y el botón **+** que abre la página Nueva sesión. Las filas archivadas permanecen en línea, atenuadas y con un glifo de archivo; no contribuyen al estado no leído ni de atención y quedan fuera de la promoción de linaje. Abrir una sesión desplaza el resaltado de selección sin reordenar las filas. Las sesiones principales con ejecuciones secundarias recientes muestran un control de despliegue y el número de sesiones secundarias; expándalo para consultar las sesiones secundarias anidadas, su estado activo o terminal y el entorno de ejecución sin salir de la barra lateral. Al seleccionar una sesión secundaria, se abre su chat y se revela automáticamente la ruta de sus antecesores. Las filas secundarias permanecen fuera de la agrupación raíz, la fijación, el arrastre, la selección múltiple y la paginación; las zonas contraídas no consumen el límite de la página visible. Las sesiones con actividad nueva desde su última lectura muestran un punto de no leído y, al abrir una, se marca como leída. Un agente también puede publicar una breve línea de estado con vencimiento y solicitar atención opcionalmente mediante un icono ámbar seleccionado; esa declaración se borra al abrir la sesión, enviar el siguiente mensaje, borrarla explícitamente o cuando vence su TTL. Los estados del ciclo de vida de los trabajadores en la nube utilizan una insignia de globo; las sesiones locales y recuperadas omiten la insignia de ubicación porque la ejecución local es la opción predeterminada. Cada fila de sesión raíz tiene un menú contextual (botón de puntos verticales o clic derecho) con Fijar/Desfijar, Marcar como no leída/leída, Cambiar nombre, Bifurcar, Mover al grupo (incluidos Nuevo grupo y Eliminar del grupo), Archivar o Desarchivar y Eliminar; los diseños táctiles mantienen visibles los controles directos de fijación y menú. Cmd/Ctrl-clic alterna las filas raíz en una selección múltiple y Mayús-clic la amplía siguiendo el orden visible; al abrir el menú en una fila seleccionada, se ofrecen acciones por lotes (Marcar N como no leídas/leídas, Mover N al grupo, Archivar N, Eliminar N) que se aplican a todas las sesiones seleccionadas, con una única confirmación para la eliminación por lotes. Arrastre una sesión raíz a **Fijadas** para fijarla o a un grupo personalizado para moverla. Los encabezados de grupos personalizados pueden contraerse, expandirse o arrastrarse para reordenarlos; los nombres de los grupos y su orden se almacenan en el Gateway (`sessions.groups.*`), por lo que se mantienen en todos los navegadores, mientras que el estado contraído permanece en el perfil del navegador. Los encabezados de grupo también tienen un menú (botón de puntos verticales o clic derecho) con Cambiar nombre del grupo, Nuevo grupo y Eliminar grupo; cambiar el nombre de un grupo o eliminarlo actualiza todas las sesiones que lo integran en el servidor, incluidas las archivadas, y al eliminar un grupo se conservan sus sesiones y se devuelven a Hilos.

## Página de nueva sesión

El botón **+** del encabezado de la lista de sesiones de la barra lateral abre un borrador a página completa en `/new`: no se crea nada hasta que se envía el primer mensaje. Un selector unificado de **Ubicación** elige la carpeta de trabajo y, para los operadores administradores, el destino de ejecución: **Gateway · local**, un Node emparejado que expone `system.run` o un perfil de nube disponible. La carpeta predeterminada es el espacio de trabajo del agente; otra ruta absoluta del Gateway requiere `operator.admin`, pero puede ejecutarse directamente sin ser un repositorio de Git. Cuando la carpeta seleccionada del Gateway es un repositorio de Git, el mismo selector ofrece aislamiento opcional mediante **Árbol de trabajo**, con un selector de rama base respaldado por `worktrees.branches` (sin recuperación) y un nombre opcional para el árbol de trabajo (la rama pasa a ser `openclaw/<name>`). Los trabajadores en la nube requieren esa ruta de árbol de trabajo gestionado; los Nodes emparejados nunca la exponen. El pie del editor elige el modelo y el nivel de razonamiento de la nueva sesión. Su control **Incógnito** crea un hilo exclusivo de la web cuya entrada de sesión, transcripción y estado de Compaction permanecen en memoria hasta que se reinicia el Gateway; OpenClaw también omite su volcado automático de memoria. El agente conserva sus herramientas normales, por lo que una solicitud explícita de guardado o una escritura de archivos mediante herramientas aún puede conservar datos. El proveedor del modelo sigue procesando los mensajes y se continúan registrando metadatos de auditoría sin contenido. Los inicios en la nube conservan las opciones de modelo y razonamiento antes de enviar la sesión a su trabajador.

En los gateways multiusuario, solo las conexiones con alcance de administrador pueden crear o ver hilos de incógnito, y las demás sesiones no pueden acceder a ellos mediante las herramientas de sesión del agente ni la búsqueda de transcripciones. El modo incógnito protege frente al almacenamiento y otros usuarios mediados por el Gateway, no frente al propietario del Gateway ni al operador del proceso, quienes siempre pueden observar las sesiones activas.

**Explorar carpetas** abre el explorador de directorios en línea del selector Ubicación, respaldado por el método `fs.listDir` exclusivo para administradores y limitado al Gateway o Node seleccionado. El Gateway y los Nodes con capacidad de exploración enumeran su sistema de archivos; un Node con capacidad de ejecución sin `fs.listDir` sigue aceptando una ruta absoluta escrita. Las ubicaciones recientes pueden restaurar conjuntamente una carpeta y su Node propietario sin transferir rutas entre hosts. Al enviar, se llama a `sessions.create` con el primer mensaje, por lo que la ejecución comienza en el mismo viaje de ida y vuelta y la interfaz salta al chat de la nueva sesión. Si el Gateway crea la sesión, pero rechaza ese primer envío, el chat conserva el mensaje y el error tras las recargas; **Reintentar** lo envía mediante la sesión ya creada en lugar de crear otra.

Dentro de **Configuración**, la barra lateral específica incluye **Preguntar a OpenClaw** y comienza con un campo **Buscar en la configuración** para encontrar rápidamente las secciones de configuración.

En la web de escritorio, un grupo fijo de controles en la parte superior izquierda del área de contenido —el equivalente web de la franja de la barra de título de macOS— contiene el control para contraer la barra lateral (⌘B) y el botón de búsqueda de la paleta de comandos (⌘K). Al hacer clic en la fila de identidad del agente en la parte superior de la barra lateral, se abre el menú del agente; **Inicio** abre la sesión principal. Cuando algo requiere atención —trabajos Cron fallidos o vencidos, autenticación del modelo próxima a caducar o caducada— aparecen indicadores de atención compactos sobre el pie de la barra lateral que enlazan con la página responsable. La fila de identidad muestra el avatar del agente (imagen de identidad o emoji), el nombre, el punto de conexión y un subtítulo en tiempo real. Su menú específico del agente contiene el selector de agentes integrado (configuraciones multiagente), **Nuevo agente**, «¿Qué puede hacer este agente?» y **Configuración del agente**. Las listas de más de diez agentes incluyen un campo de filtro y muestran primero los agentes fijados; los agentes se pueden fijar o desfijar desde la página de configuración de Agentes, y el conjunto fijado se almacena en el perfil del navegador. Al elegir un agente, Chat, Uso, Automatizaciones, Tareas, Panel de trabajo y Sesiones quedan limitados a ese agente. Cada página con ámbito específico ofrece un control **Agente** con **Todos los agentes** como vía de salida; esto amplía el ámbito de la página compartida sin cambiar el agente concreto del chat, mientras que los enlaces directos a sesiones siguen abriendo su destino. La página de configuración de Agentes conserva su propia selección `?agent=` y no sigue el ámbito compartido de la página. El pie es una única tarjeta de identidad de ancho completo que permanece disponible sin conexión y muestra **Reconectando…** debajo del último nombre de cuenta conocido. Abre el menú de la aplicación y la cuenta, cuyo encabezado de identidad del perfil va seguido de **Configuración**, **Uso**, vinculación móvil, **Obtener las aplicaciones**, **Ayuda** (ayuda, Discord, Documentación y el registro de cambios), una acción para reintentar sin conexión cuando sea necesario, el indicador de versión y compilación y el selector del modo de color. El indicador de compilación abre la página Acerca de. Cuando el Gateway se ejecuta desde un repositorio de código fuente en una rama distinta de `main`, el pie también muestra el nombre de esa rama en rojo para que resulte evidente de un vistazo que el Gateway no corresponde a una versión publicada (las instalaciones de versiones publicadas nunca lo muestran). Mayús-Comando-Coma en plataformas Apple o Ctrl-Mayús-Coma en otras plataformas abre **Configuración** sin sustituir el atajo simple Comando-Coma del navegador. Al contraer la barra lateral (⌘B o el control del grupo), se oculta por completo para ofrecer un espacio de trabajo de ancho completo; mientras está contraída, el grupo superior izquierdo conserva el control para expandirla y la búsqueda, y añade un botón de conversación nueva, reflejando lo que la aplicación de macOS aloja de forma nativa en su barra de título. La barra lateral es el único elemento de navegación en el escritorio, sin barra superior. En ventanas estrechas, la barra lateral se sustituye por un panel deslizante tras una fila de encabezado compacta que contiene el control del panel, la marca y la búsqueda de la paleta de comandos; en teléfonos, Chat integra esa fila de navegación en su barra de título, con los controles de menú y búsqueda junto al título de la sesión. En la aplicación de macOS, la fila de encabezado independiente incorpora el espacio libre de la barra de título en una única franja compacta junto a los controles de la ventana. La navegación utiliza el historial normal del navegador, por lo que los botones de retroceso y avance permiten recorrerla; la aplicación de macOS añade un control nativo de la barra lateral junto a los controles de la ventana, además de gestos de deslizamiento en el panel táctil, con botones de retroceso y avance en el borde derecho de la barra lateral cuando está expandida, y botones nativos de búsqueda (paleta de comandos) y nueva sesión cuando está contraída.

Las aprobaciones pendientes también generan un indicador de atención sobre el pie de la barra lateral;
selecciónelo para abrir la página de Aprobaciones responsable.

## Qué puede hacer (hoy)

<AccordionGroup>
  <Accordion title="Chat y conversación">
    - Chatee con el modelo mediante Gateway WS (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`). Las sesiones archivadas mantienen deshabilitado el cuadro de redacción y muestran un aviso con la acción **Desarchivar** antes de que pueda continuar la conversación.
    - Las actualizaciones del historial de Chat solicitan una ventana reciente limitada con límites de texto por mensaje, de modo que las sesiones grandes no obliguen al navegador a renderizar la carga completa de la transcripción antes de que Chat pueda utilizarse.
    - Al pasar el cursor o enfocar con el teclado un enlace público a una incidencia o un pull request de GitHub, se muestran su estado, título, autor, actividad reciente, comentarios y estadísticas de cambios. El Gateway conectado obtiene y almacena en caché los metadatos públicos sin cambiar el destino del enlace, incluso cuando la interfaz utiliza un Gateway remoto. El Gateway utiliza `GH_TOKEN` o `GITHUB_TOKEN` cuando están disponibles, tras confirmar que el repositorio es público; de lo contrario, utiliza la API anónima de GitHub con una caché más prolongada.
    - Converse mediante sesiones en tiempo real del navegador. OpenAI utiliza WebRTC directo, Google Live utiliza un token restringido de un solo uso para el navegador mediante WebSocket y los plugins de voz en tiempo real exclusivos del backend utilizan el transporte de retransmisión del Gateway. Las sesiones del navegador con capacidad de vídeo pueden elegir una cámara local del dispositivo en Configuración o cambiar de cámara desde la vista previa en directo; el navegador captura fotogramas JPEG para el proveedor en tiempo real sin transmitir el vídeo de la cámara a través del Gateway. Las sesiones del proveedor controladas por el cliente comienzan con `talk.client.create`; las sesiones de retransmisión del Gateway comienzan con `talk.session.create`. La retransmisión mantiene las credenciales del proveedor en el Gateway mientras el navegador transmite el PCM del micrófono mediante `talk.session.appendAudio`, reenvía las llamadas a herramientas del proveedor `openclaw_agent_consult` mediante `talk.client.toolCall` para aplicar la política del Gateway y utilizar el modelo OpenClaw configurado de mayor tamaño, y dirige el control por voz de la ejecución activa mediante `talk.client.steer` o `talk.session.steer`.
    - Transmita llamadas a herramientas y tarjetas con resultados de herramientas en tiempo real en Chat (eventos del agente). La actividad de las herramientas se representa como filas adaptadas a cada tipo: los comandos del shell muestran el comando con resaltado de sintaxis y una salida de estilo terminal; las llamadas compatibles de edición y escritura muestran diferencias integradas limitadas, números de línea cuando están disponibles y estadísticas de `+added -removed`; y las llamadas consecutivas se contraen en un resumen como «Se ejecutaron 13 comandos, se leyeron 6 archivos y se editaron 9 archivos». Mientras una ejecución está activa, el nombre de la llamada en curso más reciente aparece en el encabezado del grupo. Expanda una fila para inspeccionar los argumentos restantes y la salida sin procesar.
    - Títulos opcionales de propósito generados por IA para llamadas complejas a herramientas (comandos largos del shell y herramientas de plugins con muchos argumentos), habilitados mediante `gateway.controlUi.toolTitles: true` (desactivados de forma predeterminada). Los títulos proceden del método por lotes `chat.toolTitles` mediante el enrutamiento estándar de modelos auxiliares: un `utilityModel` explícito (proveedor elegido por el operador, como en otras tareas auxiliares) o, en su defecto, el modelo pequeño predeterminado declarado por el proveedor de la sesión, y se almacenan en caché en el Gateway por agente. Cuando la opción voluntaria está desactivada o no se puede utilizar ningún modelo económico, las filas conservan sus etiquetas deterministas y no se realiza ninguna llamada al modelo.
    - Inicie o descarte tareas de seguimiento efímeras sugeridas por el modelo; las sugerencias aceptadas abren una sesión nueva en un árbol de trabajo administrado con la instrucción propuesta.
    - Pestaña Actividad con resúmenes locales del navegador, que priorizan la ocultación de datos, sobre la actividad de las herramientas en tiempo real procedente de la entrega existente de `session.tool` y eventos de herramientas.

  </Accordion>
  <Accordion title="Canales, sesiones y memoria">
    - Canales: estado de los canales integrados y de los canales de plugins incluidos o externos, inicio de sesión mediante QR y configuración por canal (`channels.status`, `web.login.*`, `config.patch`).
    - Las actualizaciones de comprobación de los canales mantienen visible la instantánea anterior mientras finalizan las comprobaciones lentas de los proveedores y etiquetan las instantáneas parciales cuando una comprobación o auditoría supera el tiempo asignado por la interfaz.
    - Conversaciones (una página del espacio de trabajo en `/sessions`, con una pestaña **Árboles de trabajo** junto a ella): enumera de forma predeterminada las sesiones de los agentes configurados, permite fijar sesiones frecuentes, cambiarles el nombre, archivar o restaurar sesiones inactivas, recurrir a alternativas cuando las claves de sesiones de agentes no configurados están obsoletas y aplicar ajustes por sesión del modelo, pensamiento, modo rápido, nivel de detalle, seguimiento y razonamiento (`sessions.list`, `sessions.patch`). Un filtro de tres opciones, **Activas / Archivadas / Todas**, controla tanto esta página como la barra lateral; Todas atenúa las filas archivadas y las etiqueta explícitamente. Las sesiones archivadas conservan sus transcripciones, nunca se depuran automáticamente y permanecen apartadas hasta que se desarchivan o eliminan explícitamente. Las filas muestran un punto de no leído en las sesiones activas con actividad posterior a su última lectura, con acciones para marcar como no leída o leída (`sessions.patch { unread }`), y una acción Bifurcar que ramifica la transcripción en una sesión nueva (`sessions.create { parentSessionKey, fork: true }`). Los mosaicos de resumen situados sobre la tabla resumen la lista cargada (cantidad de sesiones, ejecuciones activas, sesiones no leídas, total de tokens y cantidad de sesiones archivadas cuando está disponible); cada fila incluye un glifo de tipo con un punto de ejecución activa, el estado se representa como un punto simple más una etiqueta y la columna Tokens muestra un medidor de uso de la ventana de contexto cuando la sesión informa del número de tokens y del tamaño del contexto. Las acciones de administración de las filas se encuentran en un menú por fila (botón de tres puntos verticales o clic derecho) que refleja el menú de sesiones de la barra lateral, y el panel de la fila incluye el entorno de ejecución del agente y la duración de la ejecución junto con los demás detalles de la sesión.
    - Los catálogos nativos de Claude y Codex de la barra lateral transmiten un host a la vez y después se concilian tras los cambios de conectividad del Node, al enfocar la página y, como máximo, cada 30 segundos mientras están visibles. Los cambios del catálogo activan una pasada de seguimiento más rápida, por lo que las sesiones creadas en las herramientas nativas aparecen sin recargar la interfaz de control. Las filas de Claude Desktop también conservan la etiqueta de su grupo personalizado local cuando existe; OpenClaw lee esa asignación del almacén local de Desktop y nunca escribe en él.
    - Agrupación de sesiones: un control Agrupar por organiza la tabla de sesiones en secciones por grupos personalizados, canal, tipo, agente o fecha. Los grupos personalizados se conservan por sesión mediante `sessions.patch` (`category`), por lo que también pueden categorizarse las sesiones iniciadas desde canales de mensajería (Discord, Telegram, WhatsApp, ...); asigne grupos arrastrando filas a una sección o mediante el selector de grupo de cada fila, y cree grupos con la acción Nuevo grupo.
    - Memoria (una pestaña de la página Agentes, limitada al agente seleccionado): estado de Dreaming, control para habilitarlo o deshabilitarlo y lector del Diario de sueños (`doctor.memory.status`, `doctor.memory.dreamDiary`, `config.patch`).
    - Importar memoria (`/memory-import`, accesible desde la pestaña Memoria de la página Agentes): obtenga una vista previa y copie la memoria automática local de Claude Code, la memoria consolidada de Codex o los archivos de memoria de Hermes en el espacio de trabajo del agente seleccionado (`migrations.memory.plan`, `migrations.memory.apply`).
    - Oferta de memoria durante la incorporación: cuando la interfaz de control se abre en modo de incorporación (`?onboarding=1`, utilizado por la aplicación complementaria de Linux tras su instalación inicial), un diálogo de una página ofrece importar las memorias detectadas con el mismo flujo de planificación y aplicación; si se omite, la página de configuración queda disponible como punto de acceso posterior.

  </Accordion>
  <Accordion title="Cron, tareas, plugins, skills, dispositivos, aprobaciones de ejecución">
    - Automatizaciones (tareas Cron): tarjetas de estadísticas (cantidad de automatizaciones, cantidad de fallos, estado del programador, próxima activación) sobre un selector de pestañas Automatizaciones/Historial de ejecuciones; la pestaña Automatizaciones enumera las tareas en una tabla filtrable (Todas/Activas/En pausa, búsqueda, filtros de programación y última ejecución, menú de acciones por fila) con sugerencias iniciales debajo, y la pestaña Historial de ejecuciones muestra las ejecuciones recientes de todas las automatizaciones (`cron.*`).
    - Tareas: registro en vivo de las tareas en segundo plano activas y recientes, con sesiones vinculadas y cancelación (`tasks.*`). El panel Tareas en segundo plano del chat agrupa el trabajo en curso y finalizado; seleccione una fila para inspeccionar su prompt acotado y el resumen de su salida o error.
    - Plugins: explore el inventario instalado y la tienda seleccionada, busque en ClawHub, instale y elimine código de plugins, y active o desactive los plugins instalados (`plugins.*`); las filas de servidores MCP permiten editar `mcp.servers` mediante los métodos de configuración.
    - Skills: estado, activación/desactivación, instalación y actualizaciones de claves de API (`skills.*`).
    - Dispositivos: un único inventario reúne los registros de dispositivos emparejados, el catálogo de nodos y la presencia en vivo (`device.pair.list`, `node.list`, `system-presence`). El host del Gateway aparece fijado en primer lugar; los clientes emparejados muestran el estado de conexión, los roles, los tokens, las capacidades y los comandos. Los emparejamientos duplicados se agrupan en un grupo desplegable, y **Limpiar N obsoletos** elimina en bloque los duplicados sin conexión confirmados por un administrador que se aprobaron automáticamente (local silencioso, CIDR de confianza o verificación mediante SSH) o que son anteriores a la procedencia de aprobación. Las entradas se pueden eliminar (`node.pair.remove`, `device.pair.remove`), el emparejamiento de dispositivos y las nuevas aprobaciones de nodos se gestionan en línea (`device.pair.*`, `node.pair.approve`/`reject`), y los códigos de configuración para móviles se crean desde la misma tarjeta.
    - Aprobaciones de ejecución: edite las listas de permitidos del gateway o del nodo y la política de consulta de `exec host=gateway/node` (`exec.approvals.*`).

  </Accordion>
  <Accordion title="Configuración">
    - Vea/edite `~/.openclaw/openclaw.json` (`config.get`, `config.set`).
    - La navegación de Configuración comienza con Preguntar a OpenClaw y luego agrupa las páginas por prioridad: General, Apariencia y Notificaciones en la parte superior; Conexiones (Conexión, Canales, Comunicaciones, Dispositivos); Agentes y herramientas (Agentes, IA y agentes, Proveedores de modelos, MCP, Automatización, Laboratorios); Privacidad y seguridad (Seguridad, Aprobaciones); y Sistema (Infraestructura, Avanzado, Depuración, Registros, Acerca de). General es un centro sencillo con los valores predeterminados de los modelos, el idioma y las estadísticas del host del gateway; cada uno de los demás ajustes se encuentra exactamente en una página.
    - Privacidad y seguridad: filas seleccionadas para la autenticación del gateway, la política de ejecución, la habilitación del navegador, el perfil de herramientas, la autenticación de dispositivos y el emparejamiento móvil, sobre las secciones respaldadas por esquemas `security`/`approvals`.
    - Aprobaciones incluye un historial de 30 días, ordenado del más reciente al más antiguo, de las solicitudes resueltas de ejecución, plugins y agentes del sistema. Filtre por tipo o recorra las filas más antiguas para revisar la decisión, el motivo, la sesión de origen y la atribución del responsable de la resolución registrada por el Gateway.
    - Laboratorios presenta los interruptores experimentales publicados. Modo de código y Enjambre son las opciones actuales y guardan `tools.codeMode.enabled` y `tools.swarm.enabled` inmediatamente; los experimentos no publicados no aparecen ni escriben claves de configuración especulativas.
    - Notificaciones: estado de las notificaciones push web del navegador, suscripción/cancelación de la suscripción y envío de prueba.
    - Avanzado: todas las secciones de configuración sin una ubicación seleccionada, además del editor JSON5 sin procesar (anteriormente, el modo Avanzado de la página General).
    - Configuración de modelos (`/settings/model-setup`) es una subpágina de Proveedores de modelos, a la que se accede desde su encabezado.
    - Agentes: una página de configuración (**Configuración → Agentes**, `/settings/agents`) con pestañas por agente (Resumen, Archivos, Herramientas, Skills, Canales, Automatizaciones, Memoria). La pestaña Resumen permite editar la identidad del agente —nombre para mostrar, emoji y una imagen de avatar que se reduce y limita en tamaño en el navegador antes de `agents.update`. Al guardar, se almacenan los campos de identidad configurados y se reflejan en `IDENTITY.md` del espacio de trabajo; los valores configurados tienen prioridad sobre las ediciones manuales de los mismos campos del archivo.
    - Perfil: una página de configuración que muestra la identidad del agente predeterminado con estadísticas de uso históricas: tokens totales, día de mayor actividad, sesión más larga, rachas de actividad, un mapa de calor anual de tokens, herramientas más utilizadas y aspectos destacados de los canales (`usage.cost`, `sessions.usage`).
    - MCP tiene una página de configuración específica con filas de servidores (transporte, habilitación y resúmenes de OAuth, filtros y paralelismo), controles directos para añadir, activar, desactivar y eliminar, comandos habituales para operadores y el editor de configuración con ámbito `mcp`. La página Plugins sigue siendo el lugar principal para conectores de un solo clic y detección.
    - Proveedores de modelos: una página de configuración que enumera todos los proveedores de modelos configurados con su icono de marca, estado de autenticación (`models.authStatus`), disponibilidad de modelos (`models.list`), datos en vivo sobre el plan, la cuota y la facturación cuando el proveedor los comunica (`usage.status`), y el gasto de las sesiones locales durante los últimos 30 días (`sessions.usage`). La acción Actualizar vuelve a consultar el estado de las credenciales y el uso del proveedor.
    - Conexión: una página de configuración (en **Conexiones**) que gestiona el enlace del propio panel con el gateway: URL de WebSocket, token del gateway, contraseña y clave de sesión predeterminada, además de la instantánea del protocolo de enlace más reciente (estado, tiempo de actividad, intervalo de pulsos y última actualización de canales). La pantalla de inicio de sesión sin conexión gestiona el caso de desconexión; esta página permite editar la conexión mientras está conectado.
    - Aplique y reinicie con validación (`config.apply`) y, después, reactive la última sesión activa.
    - Las escrituras incluyen una protección mediante hash base para evitar sobrescribir ediciones simultáneas.
    - Las escrituras (`config.set`/`config.apply`/`config.patch`) comprueban previamente la resolución de las referencias SecretRef activas en la carga útil de configuración enviada; las referencias activas enviadas que no se puedan resolver se rechazan antes de la escritura.
    - Al guardar formularios, se descartan los marcadores de posición censurados obsoletos que no pueden restaurarse desde la configuración guardada, mientras se conservan los valores censurados que aún corresponden a secretos guardados.
    - El esquema y la representación del formulario proceden de `config.schema` / `config.schema.lookup`, incluidos `title`/`description` de los campos, las indicaciones de interfaz coincidentes, los resúmenes inmediatos de elementos secundarios, los metadatos de documentación en los nodos anidados de objetos, comodines, matrices y composiciones, además de los esquemas de plugins y canales cuando están disponibles. El editor JSON sin procesar solo está disponible cuando la instantánea permite una conversión de ida y vuelta segura en formato sin procesar; de lo contrario, la interfaz de control obliga a utilizar el modo Formulario.
    - La opción "Restablecer a lo guardado" del editor JSON sin procesar conserva la estructura creada en formato sin procesar (formato, comentarios, disposición de `$include`) en lugar de volver a representar una instantánea aplanada, por lo que las ediciones externas sobreviven a un restablecimiento cuando la instantánea permite una conversión de ida y vuelta segura.
    - Los valores de objeto SecretRef estructurados se muestran como de solo lectura en los campos de texto del formulario para evitar que se corrompan accidentalmente al convertirlos de objeto a cadena.

  </Accordion>
  <Accordion title="Uso">
    - El análisis de tokens y costes estimados derivado de las sesiones se mantiene separado de la facturación del proveedor.
    - Las tarjetas de proveedores llaman a `usage.status` y muestran los nombres de los planes en vivo, los periodos de cuota, los saldos, los gastos y los presupuestos comunicados por los plugins de proveedores configurados.
    - Un fallo en el uso de un proveedor no bloquea el panel de sesiones/costes; las tarjetas de proveedores no disponibles muestran su propio estado de error.

  </Accordion>
  <Accordion title="Depuración, registros, actualización">
    - Depuración: instantáneas de estado, salud y modelos, registro de eventos y llamadas RPC manuales (`status`, `health`, `models.list`).
    - El registro de eventos incluye los tiempos de actualización y RPC de la interfaz de control, los tiempos de representación lentos del chat y la configuración, y las entradas de capacidad de respuesta del navegador correspondientes a fotogramas de animación prolongados o tareas largas cuando el navegador expone esos tipos de entrada de PerformanceObserver.
    - Registros: seguimiento en vivo de los registros de archivos del gateway con filtrado/exportación (`logs.tail`).
    - Actualización: ejecute una actualización de paquetes/git y reinicie (`update.run`) con un informe de reinicio; después, consulte periódicamente `update.status` tras volver a conectarse para verificar la versión del gateway en ejecución.

  </Accordion>
  <Accordion title="Notas del panel de automatizaciones">
    - Al seleccionar una fila, se abre una vista de detalles a página completa con un selector Activa/En pausa y Ejecutar ahora en el encabezado (ejecutar si corresponde, clonar y eliminar en su menú); la pestaña Configuración permite editar la automatización en línea (prompt, detalles, frecuencia y anulaciones avanzadas), y la pestaña Historial de ejecuciones muestra las ejecuciones de esa automatización.
    - Las automatizaciones iniciales situadas bajo la tabla rellenan previamente el formulario de creación con un prompt y una programación editables.
    - Para las tareas aisladas, la entrega utiliza de forma predeterminada el resumen de anuncio; cambie a ninguna para las ejecuciones exclusivamente internas.
    - Los campos de canal/destino aparecen cuando se selecciona el anuncio.
    - El modo Webhook utiliza `delivery.mode = "webhook"` con `delivery.to` establecido en una URL de Webhook HTTP(S) válida.
    - Para las tareas de la sesión principal, están disponibles los modos de entrega Webhook y ninguna.
    - Los controles de edición avanzada incluyen eliminar después de la ejecución, borrar la anulación del agente, opciones exactas/escalonadas de Cron, anulaciones del modelo/razonamiento del agente y conmutadores de entrega según el mejor esfuerzo.
    - La validación del formulario aparece en línea con errores por campo; los valores no válidos desactivan el botón de guardar hasta que se corrijan.
    - Establezca `cron.webhookToken` para enviar un token de portador específico; si se omite, el Webhook se envía sin una cabecera de autenticación.
    - `cron.webhook` es una alternativa heredada retirada que la validación de la configuración actual rechaza. Ejecute `openclaw doctor --fix` para migrar las tareas almacenadas que todavía usan `notify: true` a una entrega explícita por tarea mediante Webhook o al finalizar, y elimine la clave antigua.

  </Accordion>
</AccordionGroup>

## Importar memoria del asistente

Abra **Settings** → **Import Memory** para incorporar la memoria local de Codex o Claude Code
a un agente de OpenClaw. El Gateway detecta por sí mismo la memoria local compatible en su
host, por lo que una interfaz de control remota importa desde el equipo del Gateway, no desde el
equipo del navegador.

1. Elija el agente de destino.
2. Revise las colecciones de origen detectadas y los nombres de archivo Markdown. El contenido de los archivos
   no se envía en la respuesta del plan ni se muestra en la página.
3. Seleccione las colecciones que se importarán y confirme. La aplicación vuelve a generar el plan antes
   de escribir, de modo que las selecciones obsoletas fallen de forma segura.
4. Si los archivos ya existen, active **Replace existing imports**, actualice la
   vista previa y confirme el reemplazo.

Codex importa únicamente sus archivos consolidados `MEMORY.md` y `memory_summary.md`. Claude
Code importa Markdown desde los directorios de memoria automática de los proyectos y un
`autoMemoryDirectory` configurado; no importa sesiones, ajustes, instrucciones ni
credenciales mediante esta página. Los archivos se copian bajo `memory/imports/` en el
espacio de trabajo seleccionado, donde el plugin de memoria activo puede indexarlos. Las fuentes
nunca se modifican.

La planificación y la aplicación requieren `operator.admin`. Cada aplicación crea una copia de seguridad
verificada de OpenClaw cuando existe un estado, escribe un informe de migración censurado y conserva
copias de seguridad por elemento antes de reemplazar archivos de destino existentes. Consulte
[Descripción general de la memoria](/es/concepts/memory#import-from-coding-assistants) para conocer las rutas y
el comportamiento de recuperación.

## Página de MCP

La página específica de MCP es una vista para operadores de los servidores MCP gestionados por OpenClaw en `mcp.servers`. No inicia los transportes MCP por sí misma; utilícela para inspeccionar y editar la configuración guardada y, después, use `openclaw mcp doctor --probe` cuando necesite una prueba del servidor en vivo.

Flujo de trabajo habitual:

1. Abra **MCP** desde la barra lateral.
2. Consulte las tarjetas de resumen para ver los recuentos de servidores totales, habilitados, con OAuth y filtrados.
3. Revise cada fila de servidor para comprobar el transporte, la habilitación, la autenticación, los filtros, los tiempos de espera y las sugerencias de comandos.
4. Añada, habilite, deshabilite o elimine servidores directamente en la página MCP. Elija explícitamente Streamable HTTP, SSE o stdio; las líneas de comandos stdio aceptan argumentos entre comillas, como rutas con espacios. Use la página **Plugins** para conectores y detección con un solo clic.
5. Edite la sección de configuración `mcp` con ámbito definido para campos avanzados del servidor, como variables de entorno, directorios de trabajo, encabezados, rutas TLS/mTLS, metadatos de OAuth, filtros de herramientas y metadatos de proyección de Codex.
6. Use **Guardar** para escribir la configuración o **Guardar y publicar** cuando el Gateway en ejecución deba aplicar la configuración modificada.
7. Ejecute `openclaw mcp status --verbose`, `openclaw mcp doctor --probe` o `openclaw mcp reload` desde un terminal para realizar diagnósticos estáticos, pruebas en vivo o descartar el entorno de ejecución almacenado en caché.

Antes de mostrar la página, se ocultan los valores similares a URL que contienen credenciales y se escriben entre comillas los nombres de servidor en los fragmentos de comandos, para que los comandos copiados sigan funcionando con espacios o metacaracteres del shell. Referencia completa de la CLI y la configuración: [MCP](/es/cli/mcp).

## Pestaña Actividad

La pestaña Actividad se encuentra en **Configuración › Sistema**, junto a Registros y Depuración. Es un observador efímero y local del navegador para la actividad de herramientas en vivo, derivado del mismo flujo de eventos de herramientas y del Gateway `session.tool` que alimenta las tarjetas de herramientas del chat. No añade otra familia de eventos del Gateway, endpoint, almacén de actividad persistente, flujo de métricas ni flujo de observación externo.

Las entradas de Actividad solo conservan resúmenes saneados y vistas previas de salida ocultas y truncadas. Los valores de los argumentos de las herramientas no se almacenan en el estado de Actividad; la interfaz indica que los argumentos están ocultos y registra únicamente el número de campos de argumentos. La lista en memoria corresponde a la pestaña actual del navegador, se conserva al navegar dentro de la interfaz de control y se restablece al recargar la página, cambiar de sesión o pulsar **Borrar**.

## Terminal del operador

El terminal acoplable del operador está deshabilitado de forma predeterminada. Para habilitarlo, establezca `gateway.terminal.enabled: true` y reinicie el Gateway. El terminal requiere una conexión `operator.admin` y abre un PTY del host en el espacio de trabajo del agente activo. Las pestañas nuevas siguen al agente de chat seleccionado actualmente.

<Warning>
El terminal es un shell de host sin confinamiento y hereda el entorno del proceso del Gateway. Habilítelo únicamente en implementaciones con operadores de confianza. OpenClaw rechaza las sesiones de terminal para agentes con `sandbox.mode: "all"`; cambiar un agente activo a ese modo cierra sus sesiones de terminal existentes y en curso.
</Warning>

Use **Ctrl + acento grave** para mostrar u ocultar el panel acoplable. El diseño permite acoplarlo en la parte inferior o derecha, cambia de tamaño con el área visible del navegador y mantiene varias pestañas de shell. Consulte [Configuración del Gateway](/es/gateway/configuration-reference#gateway) para obtener información sobre `gateway.terminal.enabled` y la anulación opcional `gateway.terminal.shell`.

Los agentes autorizados por el propietario y sin entorno aislado pueden usar la herramienta `terminal` para trabajos prolongados o interactivos que el operador deba observar. Cada llamada a la herramienta puede abrir, leer, escribir, cambiar de tamaño, cerrar o enumerar los PTY del Gateway propios del agente. Las sesiones nuevas abren de forma predeterminada una pestaña de la interfaz de control conectada simultáneamente, para que el agente y el operador compartan la salida y cualquiera pueda escribir o cambiar el tamaño. El acceso del agente se limita exactamente a la sesión: un agente no puede leer ni controlar terminales creados por el operador ni terminales abiertos por otra sesión de agente.

Arrastre uno o más archivos al terminal activo o use el botón del clip para elegirlos. OpenClaw prepara cada archivo en la máquina propietaria del PTY y pega en el cursor las rutas absolutas entrecomilladas para el shell; nunca pulsa Intro ni ejecuta la entrada. Un indicador compacto del lote muestra el archivo actual y el número de archivos completados. Cancelar detiene el resto del lote sin pegar rutas; una transferencia fallida permanece visible para poder reintentar desde ese archivo sin volver a cargar los archivos completados. Se aceptan imágenes, PDF, archivos comprimidos y otros tipos de archivo de hasta 16 MiB por archivo. Los archivos preparados usan un directorio temporal privado del sistema en hosts POSIX (modo del directorio `0700`, modo del archivo `0600`) o un directorio dentro del límite de la ACL del perfil de usuario en Windows, además de un temporizador de limpieza de 24 horas; mueva o copie todo aquello que necesite conservar.

La inserción de rutas es compatible con PowerShell, `cmd.exe` y shells POSIX reconocidos (`sh`, Bash, Dash, Ash, Ksh, Zsh y Fish), incluido Git Bash en Windows. Se rechazan otras anulaciones del shell porque sus reglas de entrecomillado no pueden inferirse de forma segura; ejecute el Gateway dentro de WSL para disponer de un terminal WSL nativo y rutas de carga de Linux. También se rechazan las rutas `cmd.exe` que contienen `%` o `!`, porque ese shell expande esos caracteres incluso dentro de comillas dobles.

Las sesiones de Codex y Claude Code detectadas en la barra lateral de sesiones pueden abrirse en su CLI nativa dentro del mismo panel de terminal. En **Configuración › Chat**, establezca **Abrir hilos de Codex/Claude en** en **Terminal** para que al hacer clic normalmente en una fila se abra `codex resume` o `claude --resume`; de forma predeterminada se sigue usando el visor de solo lectura de OpenClaw. El menú contextual o de tres puntos de cada fila siempre ofrece ambas opciones, y el encabezado del visor incluye **Abrir en el terminal** cuando la sesión cumple los requisitos.

La idoneidad se determina por sesión y por host. Las sesiones locales del Gateway inician el comando de reanudación propiedad del proveedor en el host del Gateway. Las sesiones de nodos emparejados inician un comando permitido del proveedor en el nodo propietario y retransmiten únicamente los eventos de salida, entrada y cambio de tamaño de ese PTY; esto no expone un shell general del nodo ni acepta comandos proporcionados por el navegador. Las cargas de archivos usan el comando de nodo `terminal.upload`, independiente y con tamaño limitado, y permanecen vinculadas a la sesión de terminal ya abierta. Apruebe la actualización del emparejamiento del nodo cuando ese comando aparezca por primera vez. Los nodos que no anuncian el comando correspondiente de reanudación del terminal, incluidos los puentes de procesos de trabajo integrados sin transmisión dúplex, mantienen disponible el visor e indican que no se puede abrir el terminal; los nodos antiguos aún pueden ejecutar un terminal, pero no pueden recibir archivos arrastrados.

Las sesiones propiedad de la conexión sobreviven a las desconexiones: una recarga de la página, la suspensión del portátil o una interrupción de red separa la sesión en el Gateway en lugar de finalizarla, y la misma pestaña del navegador vuelve a conectarse al restablecerse la conexión, con la salida reciente reproducida. Las sesiones separadas propiedad de la conexión se finalizan después de `gateway.terminal.detachedSessionTimeoutSeconds` (valor predeterminado: 300 segundos; `0` restaura la finalización al desconectarse). Conectarse a una de estas sesiones sigue funcionando como una toma de control al estilo de tmux.

Las sesiones propiedad del agente no están vinculadas a una conexión del navegador. `terminal.attach` añade cada navegador como visor sin asumir la propiedad, y cerrar una pestaña del visor solo desconecta ese navegador. El PTY permanece hasta que el agente propietario lo cierra, su proceso termina, una política lo deshabilita o el Gateway se apaga. `terminal.list` marca cada entrada como propiedad de la conexión o del agente, y `terminal.text` permite que una conexión de administrador lea la salida reciente en texto sin formato sin conectarse.

El terminal también está disponible como documento de pantalla completa que solo contiene el terminal en `/?view=terminal`. Las aplicaciones para iOS y Android integran esta página en sus pantallas de Terminal y reutilizan las credenciales almacenadas del Gateway; la disponibilidad depende de las mismas condiciones `gateway.terminal.enabled` y `operator.admin`, y la página muestra un aviso cuando el Gateway conectado no ofrece el terminal.

## Panel del navegador

La interfaz de control incluye un panel de navegador acoplable que representa el navegador controlado por el Gateway —el mismo que los agentes controlan mediante la [herramienta de navegador](/es/tools/browser-control)— en cualquier navegador web convencional, sin necesidad de una vista web nativa. Aparece cuando el Gateway conectado anuncia `browser.request` a una conexión `operator.admin`; el botón del globo en la barra del espacio de trabajo del hilo permite mostrarlo u ocultarlo. El panel muestra una instantánea en vivo de la página con pestañas, una barra de URL editable, controles para retroceder, avanzar y recargar, y una opción para abrirla en el navegador; se acopla a la derecha o abajo y reenvía clics, desplazamientos con la rueda y escritura básica a la página remota.

Dos modos de captura empaquetan el contexto de la página para el agente:

- **Anotar (lápiz)**: permite dibujar marcas a mano alzada sobre la página. **Enviar al chat** combina los trazos con la captura de pantalla, adjunta la imagen al editor del chat activo y rellena previamente una instrucción que describe la URL y el título de la página, así como cada región marcada, para que el agente sepa exactamente qué se ha rodeado.
- **Inspeccionar (puntero)**: pase el cursor para ver el elemento que hay debajo (selector, nombre accesible, rol y tamaño); haga clic para enviar los detalles de ese elemento y una captura de pantalla resaltada mediante el mismo flujo del editor. La inspección, el desplazamiento con la rueda y los controles de retroceso y avance requieren `browser.evaluateEnabled` (activado de forma predeterminada).

La aplicación para macOS conserva su barra lateral nativa de navegación de enlaces para los enlaces pulsados en el panel; el panel del navegador también funciona allí y permite anotar páginas en todas las demás plataformas.

## Comportamiento del chat

<AccordionGroup>
  <Accordion title="Semántica de envío e historial">
    - `chat.send` es **no bloqueante**: confirma la recepción inmediatamente con `{ runId, status: "started" }` y la respuesta se transmite mediante eventos `chat`. Los clientes de confianza de la interfaz de control también pueden recibir metadatos opcionales sobre los tiempos de confirmación para diagnósticos locales.
    - Las cargas del chat admiten imágenes y archivos que no sean de vídeo. Las imágenes conservan la ruta de imagen nativa; los demás archivos se almacenan como contenido multimedia administrado y se muestran en el historial como enlaces de archivos adjuntos.
    - Volver a enviar con el mismo `idempotencyKey` devuelve `{ status: "in_flight" }` mientras está en ejecución y `{ status: "ok" }` tras completarse.
    - Las respuestas de `chat.history` tienen un tamaño limitado para proteger la interfaz de usuario. Cuando las entradas de la transcripción son demasiado grandes, el Gateway puede truncar campos de texto largos, omitir bloques de metadatos pesados y sustituir los mensajes demasiado grandes por un marcador de posición (`[chat.history omitted: message too large]`).
    - Cuando un mensaje visible del asistente se ha truncado en `chat.history`, el lector lateral puede obtener bajo demanda la entrada completa de la transcripción normalizada para su visualización mediante `chat.message.get`, usando `sessionKey`, el `agentId` activo cuando sea necesario y el `messageId` de la transcripción. Si el Gateway sigue sin poder devolver más contenido, el lector muestra un estado explícito de indisponibilidad en lugar de repetir silenciosamente la vista previa truncada.
    - Las imágenes generadas o producidas por el asistente se conservan como referencias de contenido multimedia administrado y se vuelven a servir mediante URL de contenido multimedia autenticadas del Gateway, por lo que las recargas no dependen de que las cargas de imagen base64 sin procesar permanezcan en la respuesta del historial del chat.
    - Al renderizar `chat.history`, la interfaz de control elimina del texto visible del asistente las etiquetas de directivas insertadas que solo sirven para la visualización (por ejemplo, `[[reply_to_*]]` y `[[audio_as_voice]]`), las cargas XML de llamadas a herramientas en texto sin formato (incluidos `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y los bloques truncados de llamadas a herramientas), así como los tokens de control del modelo filtrados en formato ASCII o de ancho completo. Omite las entradas del asistente cuyo texto visible completo sea únicamente el token silencioso exacto `NO_REPLY` / `no_reply` o el token de confirmación de Heartbeat `HEARTBEAT_OK`.
    - Durante un envío activo y la actualización final del historial, la vista del chat mantiene visibles los mensajes locales optimistas del usuario y del asistente si `chat.history` devuelve brevemente una instantánea anterior; la transcripción canónica sustituye esos mensajes locales cuando el historial del Gateway se pone al día.
    - Los eventos `chat` en directo representan el estado de entrega, mientras que `chat.history` se reconstruye a partir de la transcripción persistente de la sesión. Tras los eventos finales de las herramientas, la interfaz de control vuelve a cargar el historial y combina únicamente una pequeña cola optimista; el límite de la transcripción se documenta en [WebChat](/es/web/webchat).
    - `chat.inject` añade una nota del asistente a la transcripción de la sesión y difunde un evento `chat` para actualizaciones exclusivas de la interfaz de usuario (sin ejecución del agente ni entrega al canal).
    - La barra lateral enumera todas las sesiones activas cargadas por sección de agente y en los grupos de fijadas/canal/trabajo/personalizados/Chats, con una única acción Nueva sesión que abre el cuadro de diálogo de borrador. Abrir una fila visible solo desplaza el resaltado. Las sesiones pueden soltarse en Fijadas para fijarlas, o en un grupo personalizado o Chats para moverlas; los grupos personalizados se pueden contraer y reordenar mediante arrastre, los nombres y el orden de los grupos se sincronizan mediante el Gateway y el estado contraído permanece en el navegador. Una nueva sesión del panel obtiene de forma asíncrona un título conciso generado a partir de su primer mensaje que no sea un comando; los nombres explícitos y la identidad autenticada del remitente permanecen separados, por lo que los nombres de las cuentas nunca se utilizan como títulos generados. Configure `agents.defaults.utilityModel` (o `agents.entries.*.utilityModel`) para dirigir esta llamada de modelo independiente a un modelo de menor coste; si ese modelo distinto falla, la generación del título vuelve a intentarlo una vez con el modelo principal. Al expandir la sección de otro agente, se pueden explorar las sesiones de ese agente sin salir del chat abierto.
    - La búsqueda de hilos se encuentra en la paleta de comandos (⌘K o el botón de búsqueda del grupo de controles superior izquierdo): al escribir una consulta, se recorre un número limitado de páginas coincidentes entre los agentes, se filtran las filas internas secundarias/de Cron y se muestran las coincidencias visibles junto a los comandos de navegación. La página Hilos conserva la lista exhaustiva con búsqueda y filtros.
    - Cada fila de la barra lateral mantiene acceso directo para fijarla, además de un menú contextual completo para el estado de no leído, el cambio de nombre, la bifurcación, la agrupación, el archivado y la eliminación. Las filas seleccionadas de forma múltiple (Cmd/Ctrl-clic, Mayús-clic para intervalos) disponen de un menú por lotes que incluye el estado de no leído, la agrupación, el archivado y la eliminación; el archivado y la eliminación por lotes permanecen deshabilitados a menos que todas las sesiones seleccionadas se puedan archivar. No se pueden archivar una ejecución activa ni la sesión principal de un agente. Al archivar o eliminar la sesión seleccionada actualmente, Chat vuelve a la sesión principal de ese agente.
    - En la aplicación para macOS, la marca de OpenClaw utiliza la franja vacía de la barra de título nativa situada junto a los controles de la ventana, en lugar de ocupar una fila de la barra lateral.
    - En anchos de escritorio, los controles del chat permanecen en una sola fila compacta y se contraen al desplazarse hacia abajo por la transcripción; desplazarse hacia arriba, volver al principio o llegar al final restaura los controles.
    - El encabezado de la sesión muestra un pequeño grupo de avatares junto a la etiqueta del espacio de trabajo cuando otras personas están viendo la misma sesión; enumera hasta cuatro avatares de espectadores con un recuento adicional y desaparece cuando no hay nadie más.
    - Los mensajes consecutivos duplicados que solo contienen texto se renderizan como una única burbuja con una insignia de recuento. Los mensajes que contienen imágenes, archivos adjuntos, resultados de herramientas o vistas previas de Canvas no se contraen.
    - Las burbujas de los mensajes del usuario incluyen acciones de transcripción: un botón para rebobinar al pasar el cursor (ventana emergente de confirmación con la opción "Don't ask again"), además de **Rebobinar hasta aquí** y **Bifurcar desde aquí** al hacer clic con el botón derecho. Rebobinar devuelve la sesión al estado inmediatamente anterior a ese mensaje y devuelve su texto al editor para modificarlo y reenviarlo (`sessions.rewind`, `operator.admin`); bifurcar crea una nueva sesión a partir del prefijo de la ruta activa anterior al mensaje, la abre y rellena su editor con el mismo texto (`sessions.fork`, `operator.write`). Ambas acciones se deshabilitan con una descripción emergente explicativa mientras el agente está trabajando, se aplican únicamente a mensajes de usuario persistentes y se rechazan en las sesiones cuya conversación pertenece a un entorno externo de agentes. Rebobinar solo desplaza el contexto del chat —los archivos y otros efectos secundarios de las herramientas no se revierten— y la transcripción anterior al rebobinado permanece conservada en el almacén de sesiones de solo anexado. Cuando ese almacén contiene varias ramas de transcripción, la barra de título del chat muestra un menú de ramas con el mensaje más reciente, el número de mensajes y la antigüedad de cada rama; al seleccionar una rama inactiva, la sesión actual vuelve a esa ruta conservada (`sessions.branches.list`, `operator.read`; `sessions.branches.switch`, `operator.admin`). El cambio de rama tampoco está disponible mientras el agente está trabajando, y seleccionar la rama ya activa produce un error tipado de operación nula en el límite RPC. La acción independiente de ocultar en las burbujas del usuario oculta un mensaje únicamente en el navegador actual; el mensaje permanece en la transcripción y el agente sigue viéndolo.
    - Cuando el checkout de una sesión se encuentra en una rama no predeterminada de un repositorio de GitHub, la vista del chat fija etiquetas de pull requests encima del editor: número de PR, repositorio, rama, recuentos de diferencias, un indicador de CI y el estado de borrador/fusionado/cerrado, cada uno con un enlace al PR. La fila muestra como máximo dos etiquetas —primero los PR activos (abiertos o en borrador)— y un botón "Show more" revela el historial contraído de PR fusionados o cerrados. El indicador de CI abre una pequeña ventana emergente de supervisión de CI con el recuento de comprobaciones aprobadas, fallidas, en ejecución y omitidas, además de un enlace a la página de comprobaciones del PR. La detección se ejecuta en el servidor mediante `controlUi.sessionPullRequests`, que reutiliza `GH_TOKEN`/`GITHUB_TOKEN` del Gateway cuando están configurados. Cuando se alcanza el límite de solicitudes de la API de GitHub, las etiquetas conservan el último estado conocido y muestran una advertencia de que podría estar desactualizado; descartar una etiqueta la oculta para esa sesión en el perfil actual del navegador. Antes de que exista un PR, la fila muestra la propia rama: repositorio, nombre de la rama y tamaño +/− de las diferencias respecto a la base de fusión de la rama predeterminada (trabajo confirmado y sin confirmar). Cuando la rama enviada contiene confirmaciones que se pueden comparar, la fila añade un botón Create PR que abre la página de nueva pull request de GitHub; antes de eso, una sesión con archivos modificados (confirmados, sin confirmar o sin seguimiento) sigue mostrando la fila sin el botón. La fila se oculta mientras exista un PR abierto o en borrador. La fila de la rama procede únicamente del repositorio Git local, por lo que sigue disponible mientras GitHub está limitado por la cuota e incluye la misma advertencia de estado obsoleto, ya que no se puede confiar en "no se encontró ningún PR" hasta que se restablezca el límite.
    - El panel de diferencias de la sesión muestra lo que realmente ha cambiado el checkout de una sesión: el botón de rama de la barra del espacio de trabajo o de la barra de título del chat abre el panel de detalles con las diferencias por archivo del trabajo de la rama, sin confirmar y sin seguimiento respecto a la base de fusión de la rama predeterminada del checkout: punto de estado, flecha de cambio de nombre, recuentos +/− por archivo, archivos contraíbles y marcadores de "N líneas sin modificar" entre fragmentos. Las diferencias se calculan en el servidor mediante el método `sessions.diff` del Gateway (ámbito `operator.read`); los archivos binarios y demasiado grandes se reducen a entradas que solo muestran estadísticas, y el botón solo aparece cuando el Gateway conectado anuncia `sessions.diff`.
    - Cada panel de Chat tiene una barra de título. Haga clic en el título de la sesión para cambiarle el nombre; la etiqueta del espacio de trabajo copia la ruta o rama del checkout y puede mostrar los espacios de trabajo locales del Gateway en el administrador de archivos del host. Las sesiones remotas y de nodos de ejecución conservan las acciones de copia, pero ocultan la opción de mostrar.
    - La barra del espacio de trabajo del hilo en cada panel de Chat enumera los archivos del hilo, los archivos del proyecto y los artefactos. De forma predeterminada, se acopla al borde derecho del panel; arrastre su encabezado (o utilice el botón de acoplamiento) para moverla a la parte inferior, y la elección se guarda en el perfil actual del navegador. Una barra contraída no ocupa espacio alguno: vuelva a abrirla con ⇧⌘B o con el conmutador de archivos de la barra de título, que muestra una insignia con el recuento de archivos modificados. El panel independiente de detalles de archivos, herramientas y Canvas no se ve afectado.
    - Al hacer clic en una referencia de archivo del chat, una ruta de archivo de una tarjeta expandida de herramienta de lectura/edición/escritura o una fila de archivo de la barra del espacio de trabajo, se abre el panel de detalles del archivo: una vista de código basada en CodeMirror con resaltado de sintaxis, números de línea, salto a línea, búsqueda dentro del archivo, acciones de copia y un menú para abrirlo en un editor externo. Cuando el Gateway anuncia `sessions.files.set` a una conexión `operator.admin`, el panel añade un modo Edit con seguimiento de modificaciones y guardado mediante Cmd/Ctrl-S; los borradores sin guardar sobreviven a la navegación entre archivos, paneles y sesiones en la pestaña actual del navegador hasta que se guardan o descartan explícitamente. Los guardados utilizan una operación de comparación e intercambio basada en un hash del contenido devuelto por `sessions.files.get`: si el archivo ha cambiado en el disco desde que se cargó (por ejemplo, porque el agente siguió trabajando), el panel muestra un aviso de conflicto con las acciones Reload (usar el contenido más reciente) y Overwrite (conservar la edición local). Las escrituras utilizan las mismas protecciones de seguridad del sistema de archivos del espacio de trabajo que las lecturas —contención de rutas, rechazo de enlaces simbólicos y enlaces físicos, y un límite de 256 KB en UTF-8— y solo sobrescriben archivos existentes; el editor nunca los crea ni los elimina.
    - La barra de tareas en segundo plano de cada panel de Chat enumera las tareas en segundo plano y los subagentes del agente actual (`tasks.list` limitado por agente y actualizado en directo mediante eventos `task`): el trabajo en ejecución muestra un temporizador de tiempo transcurrido en directo, el recuento de usos de herramientas, la herramienta utilizada actualmente y un control para detenerlo; la sección contraíble de tareas finalizadas añade las duraciones de las ejecuciones; y un enlace Ver transcripción abre la sesión secundaria de la tarea en el panel. Ábrala con el conmutador de actividad de la barra de título; la instantánea de las tareas se carga de forma anticipada, por lo que muestra una insignia con el recuento de tareas en ejecución sin necesidad de abrir primero la barra. La página Tareas sigue siendo el registro completo de todos los agentes.
    - La barra de espacios de trabajo, la barra de tareas en segundo plano y el panel de detalles se adaptan al ancho propio de cada panel en lugar de al de la ventana: en un panel estrecho o una ventana compacta, ambas barras se muestran como franjas inferiores (los controles de acoplamiento lateral se ocultan hasta que el panel se ensancha; la barra de espacios de trabajo tiene prioridad sobre el espacio lateral cuando solo cabe una columna), y el panel de detalles se apila debajo del hilo con un controlador de redimensionamiento horizontal en lugar de compartir la fila con él. En las áreas de visualización del tamaño de un teléfono, el panel de detalles sigue abriéndose a pantalla completa.
    - Los selectores de modelo y de razonamiento de la cabecera del chat actualizan de inmediato la sesión activa mediante `sessions.patch`; son anulaciones persistentes de la sesión, no opciones de envío aplicables únicamente a un turno.
    - **Vista dividida:** ábrala desde la barra de título del chat (junto a los conmutadores de diferencias del hilo, tareas en segundo plano y archivos del hilo) y, a continuación, divida el panel activo hacia la derecha o hacia abajo en tantos paneles como quepan. Cada panel tiene su propio hilo, transcripción, editor de mensajes y flujo de herramientas.
    - Los agentes con la herramienta `screen` pueden solicitar los mismos cambios de panel, barra lateral, terminal, navegador, foco y navegación mientras haya una interfaz de control compatible conectada. La versión 1 del protocolo aplica el comando a todas las interfaces de control compatibles conectadas; consulte [Pantalla](/es/tools/screen).
    - Arrastre una sesión desde la barra lateral hasta el chat para abrirla en un panel. Una vista previa animada del destino se desliza entre las zonas y etiqueta el resultado —«Dividir» sobre la mitad exacta que ocupará un panel nuevo, «Abrir aquí» sobre un panel completo—; también se puede soltar desde el modo de panel único.
    - El panel dividido activo controla la selección de la barra lateral y la URL. Su barra de título añade controles para dividir y cerrar; los separadores permiten redimensionar las columnas y los paneles apilados, y el navegador almacena localmente el diseño entre recargas.
    - En pantallas estrechas, la vista dividida conserva el diseño, pero solo muestra el panel activo, incluida su cabecera con el control para cerrarlo.
    - Si se envía un mensaje mientras aún se está guardando un cambio del selector de modelo para la misma sesión, el editor de mensajes espera a que se complete esa actualización de la sesión antes de llamar a `chat.send`, para que el envío utilice el modelo seleccionado.
    - Al escribir `/new`, se crea y se cambia a la misma sesión nueva del panel de control que con Nuevo chat, excepto cuando `session.dmScope: "main"` está configurado y el elemento principal actual es la sesión principal del agente; en ese caso, se restablece la sesión principal en el mismo lugar. Al escribir `/reset`, se conserva el restablecimiento explícito en el mismo lugar del Gateway para la sesión actual.
    - El selector de modelo del chat solicita la vista de modelos configurada del Gateway. Si `agents.defaults.modelPolicy.allow` no está vacío, esa política controla el selector, incluidas las entradas `provider/*` que mantienen dinámicos los catálogos específicos de cada proveedor. De lo contrario, el selector muestra las entradas configuradas y los proveedores con autenticación utilizable; los alias y ajustes de `agents.defaults.models` no lo restringen. El catálogo completo sigue estando disponible mediante el RPC de depuración `models.list` con `view: "all"`.
    - Cuando los informes recientes de uso de sesión del Gateway incluyen los tokens de contexto actuales, la barra de herramientas del editor de mensajes del chat muestra un pequeño anillo de uso del contexto con el porcentaje utilizado. Abra el anillo para consultar la ventana de contexto actual, los recuentos de tokens de la ejecución más reciente y el coste total estimado, la identidad del proveedor y del modelo, y el desglose de los costes de entrada, salida y caché de la respuesta más reciente del proveedor, cuando se proporcione. El anillo cambia al estilo de advertencia cuando la presión sobre el contexto es alta y, al alcanzar los niveles recomendados de Compaction, muestra un botón compacto que ejecuta la ruta normal de Compaction de la sesión. Las instantáneas obsoletas de tokens se ocultan hasta que el Gateway vuelve a informar de datos de uso recientes.

  </Accordion>
  <Accordion title="Modo de conversación (tiempo real en el navegador)">
    El modo de conversación utiliza un proveedor de voz en tiempo real registrado. Configure OpenAI con `talk.realtime.provider: "openai"` y un perfil de clave de API `openai`, `talk.realtime.providers.openai.apiKey` o `OPENAI_API_KEY`. OpenAI Realtime utiliza la API pública de Platform y requiere una clave de API de Platform; un inicio de sesión OAuth de Codex no satisface esta interfaz. Configure Google con `talk.realtime.provider: "google"` y `talk.realtime.providers.google.apiKey`. El navegador nunca recibe una clave de API estándar del proveedor: OpenAI recibe un secreto efímero de cliente de Realtime para WebRTC, y Google Live recibe un token de autenticación restringido y de un solo uso de la API Live para una sesión WebSocket del navegador, con las instrucciones y declaraciones de herramientas fijadas en el token por el Gateway. Los proveedores que solo ofrecen un puente de tiempo real de backend funcionan mediante el transporte de retransmisión del Gateway, por lo que las credenciales y los sockets del proveedor permanecen en el servidor mientras el audio del navegador circula mediante RPC autenticadas del Gateway. El prompt de la sesión de Realtime lo compone el Gateway; `talk.client.create` no acepta reemplazos de instrucciones proporcionados por quien realiza la llamada.

    Los valores predeterminados persistentes de proveedor, modelo, voz, transporte, esfuerzo de razonamiento, umbral exacto de VAD, duración del silencio y relleno de prefijo se encuentran en **Settings → Communications → Talk**; para cambiarlos se requiere acceso a `operator.admin`. Configurar la retransmisión del Gateway fuerza la ruta de retransmisión del backend; configurar WebRTC mantiene la sesión bajo el control del cliente y genera un error, en lugar de recurrir silenciosamente a la retransmisión, si el proveedor no puede crear una sesión del navegador.

    El control de conversación es el botón del micrófono de la barra de herramientas del cuadro de redacción. Su menú desplegable muestra **System default** y todos los micrófonos expuestos por el navegador, incluidas las entradas USB, Bluetooth y virtuales. El ID del dispositivo seleccionado permanece en el navegador y nunca se envía al Gateway; si ese dispositivo exacto desaparece, el modo de conversación solicita que se elija otra entrada en lugar de grabar silenciosamente desde otro micrófono. Mientras la conversación está activa, el botón del micrófono se convierte en una cápsula que muestra el medidor del nivel de entrada en directo; al hacer clic se detiene la entrada de voz y, al pasar el cursor, aparece el glifo de detención. Los lectores de pantalla anuncian `Connecting voice input...`, `Listening...` o `Asking OpenClaw...` mientras una llamada de herramienta en tiempo real consulta el modelo de mayor tamaño configurado mediante `talk.client.toolCall`. Detener una respuesta del agente en curso sigue siendo una acción independiente mediante el control cuadrado **Stop** situado junto a la cápsula.

    **Conversación con vídeo** está disponible para las sesiones de navegador de OpenAI Realtime WebRTC y Google Live. Haga clic en el botón de la cámara, permita el acceso a la cámara y al micrófono, y confirme la vista previa local. OpenAI envía un fotograma JPEG acotado mediante su canal de datos del navegador cuando `describe_view` solicita contexto visual. Google Live envía fotogramas JPEG acotados directamente desde el navegador al proveedor, con el máximo admitido de un fotograma por segundo, y responde a las llamadas de función `describe_view` con el estado de la transmisión de la cámara. Los fotogramas de la cámara nunca pasan por el Gateway. Al detener la conversación, se cierra la vista previa y se liberan ambas pistas multimedia. Consulte las [capacidades de la API Live](https://ai.google.dev/gemini-api/docs/live-api/capabilities#video) y la [guía de llamadas a funciones](https://ai.google.dev/gemini-api/docs/live-api/tools) de Google para conocer los contratos de comunicación del proveedor.

    Prueba de humo en directo para responsables de mantenimiento: `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` verifica el puente WebSocket de backend de OpenAI, el intercambio SDP de WebRTC de navegador de OpenAI, la configuración de navegador de Google Live con token restringido, incluido un fotograma JPEG y una ida y vuelta de la función `describe_view`, y el adaptador de navegador de retransmisión del Gateway con medios de micrófono simulados. El comando solo imprime el estado del proveedor y no registra secretos.

  </Accordion>
  <Accordion title="Detención y cancelación">
    - Haga clic en **Stop**. Las ejecuciones con un ID de ejecución local exacto llaman a `chat.abort`; cuando el estado de la sesión seleccionada indica que hay trabajo activo, pero la interfaz de control no dispone de un ID de ejecución local, se llama a `sessions.abort` en su lugar. En las sesiones no globales, esa ruta de la sesión seleccionada también descarta los seguimientos en cola para que no puedan reiniciar el trabajo después de la detención.
    - Mientras una ejecución está activa, los seguimientos normales utilizan el modo `messages.queue` efectivo del Gateway. `steer` los inyecta en el turno en curso; los demás modos mantienen la entrega en cola persistente del navegador. Si se rechaza el redireccionamiento, también se recurre a esa cola. Haga clic en **Steer** en un mensaje en cola para inyectarlo manualmente.
    - **Settings → Appearance → Chat → Follow-ups while the agent is working** permite reemplazar ese valor predeterminado del servidor para el navegador actual. La página señala explícitamente que existe un valor reemplazado y ofrece **Reset to server default**. `Steer into the active run` envía los seguimientos inmediatamente, mientras que `Queue until the run ends` los retiene hasta que finaliza la ejecución.
    - Escriba `/stop` (o frases de cancelación independientes como `stop`, `stop action`, `stop run`, `stop openclaw`, `please stop`) para cancelar fuera de banda.
    - `chat.abort` admite `{ sessionKey }` (sin `runId`) para cancelar todas las ejecuciones activas de esa sesión. La interfaz de control utiliza `sessions.abort` cuando no dispone de un ID de ejecución local.

  </Accordion>
  <Accordion title="Conservación parcial tras la cancelación">
    - Cuando se cancela una ejecución, el texto parcial del asistente aún puede mostrarse en la interfaz.
    - El Gateway conserva en el historial de la transcripción el texto parcial cancelado del asistente cuando existe una salida almacenada en el búfer.
    - Las entradas conservadas incluyen metadatos de cancelación para que los consumidores de la transcripción puedan distinguir las partes parciales canceladas de la salida de finalización normal.

  </Accordion>
</AccordionGroup>

## Pérdida de conexión y reconexión

Una vez establecida una sesión, la interrupción de la conexión con el Gateway no cierra la sesión. El panel
permanece visible con una cápsula flotante ámbar que muestra "Gateway connection lost — Reconnecting…" debajo de la barra
superior mientras el cliente vuelve a intentarlo automáticamente con espera incremental (desde 800 ms hasta 15 s). Las actualizaciones en directo y
las acciones de tiempo real o de sesión se pausan hasta que se restablece la conexión; **Retry now** en la cápsula fuerza un
intento inmediato. El chat continúa siendo editable: los envíos normales de texto y archivos adjuntos se conservan en el
almacenamiento del navegador de la pestaña actual, limitado al Gateway y la sesión, se muestran como pendientes de reconexión y se envían
automáticamente cuando vuelve el Gateway. Los controles en directo y los comandos con barra diagonal permanecen inaccesibles mientras
no haya conexión, salvo que **Stop** puede poner en cola un ID de ejecución local exacto para reproducir la acción. Una detención solo de sesión
no se reproduce porque podría comenzar un trabajo más reciente en esa sesión antes de que vuelva la conexión.

Cuando este navegador ya contiene credenciales (un token o una contraseña configurados, o un token de dispositivo
aprobado), las primeras aperturas y las recargas muestran una pequeña marca animada de OpenClaw mientras se
establece la conexión, en lugar de mostrar brevemente la pantalla de inicio de sesión. La pantalla de inicio de sesión solo aparece cuando todavía no
hay credenciales almacenadas o cuando el Gateway las rechaza activamente (token o contraseña incorrectos, emparejamiento revocado);
estos estados requieren una intervención en lugar de esperar.

## Instalación de la PWA y notificaciones web push

La interfaz de control incluye un `manifest.webmanifest` y un service worker, por lo que los navegadores modernos pueden instalarla como una PWA independiente. Web Push permite que el Gateway active la PWA instalada con notificaciones incluso cuando la pestaña o la ventana del navegador no están abiertas.

Dentro de la aplicación para macOS, la página de configuración de notificaciones muestra el permiso de notificaciones nativo de la aplicación en lugar de las notificaciones push del navegador, porque la aplicación entrega las notificaciones de forma nativa.

Si la página muestra **Protocol mismatch** justo después de una actualización de OpenClaw, vuelva a abrir primero el panel con `openclaw dashboard` y realice una actualización forzada. Si el problema persiste, borre los datos del sitio correspondientes al origen del panel o pruebe en una ventana privada del navegador; una pestaña antigua o la caché del service worker del navegador pueden seguir ejecutando un paquete de la interfaz de control anterior a la actualización contra el Gateway más reciente.

| Interfaz                                           | Función                                                                       |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `ui/public/manifest.webmanifest`                   | Manifiesto de la PWA. Los navegadores ofrecen "Install app" cuando está accesible. |
| `ui/public/sw.js`                                  | Service worker que gestiona los eventos `push` y los clics en las notificaciones. |
| `state/openclaw.sqlite` → `web_push_vapid_keys`    | Par de claves VAPID generado automáticamente para firmar las cargas útiles de Web Push. |
| `state/openclaw.sqlite` → `web_push_subscriptions` | Endpoints, claves y marcas de tiempo de registro persistentes de las suscripciones del navegador. |

Las actualizaciones desde los almacenes retirados `push/vapid-keys.json` y `push/web-push-subscriptions.json` se importan mediante `openclaw doctor --fix`. Detenga el Gateway antes de ejecutar esa reparación para que un proceso antiguo no pueda volver a crear el estado retirado durante la importación. Ejecute la reparación antes de utilizar Web Push tras una actualización; el registro, la entrega, la eliminación y la resolución de claves se niegan a continuar mientras persista cualquiera de los orígenes retirados o una reclamación interrumpida de Doctor. El entorno de ejecución del Gateway solo lee y escribe en SQLite.

Reemplace el par de claves VAPID mediante variables de entorno del proceso del Gateway cuando se necesite fijar las claves (implementaciones con varios hosts, rotación de secretos o pruebas):

- `OPENCLAW_VAPID_PUBLIC_KEY`
- `OPENCLAW_VAPID_PRIVATE_KEY`
- `OPENCLAW_VAPID_SUBJECT` (el valor predeterminado es `https://openclaw.ai`)

La interfaz de control utiliza estos métodos del Gateway restringidos por ámbito para registrar y probar las suscripciones del navegador:

- `push.web.vapidPublicKey` obtiene la clave pública VAPID activa.
- `push.web.subscribe` registra un `endpoint` junto con `keys.p256dh`/`keys.auth`.
- `push.web.unsubscribe` elimina un endpoint registrado.
- `push.web.test` envía una notificación de prueba a la suscripción de quien realiza la llamada.

<Note>
Web Push es independiente de la ruta de retransmisión APNS de iOS (consulte [Configuración](/es/gateway/configuration) para conocer las notificaciones push respaldadas por retransmisión) y del método `push.test`, que se dirige al emparejamiento móvil nativo.
</Note>

## Contenido insertado alojado

Los mensajes del asistente pueden representar contenido web alojado en línea mediante el shortcode `[embed ...]`. La política del entorno aislado del iframe se controla mediante `gateway.controlUi.embedSandbox`:

La herramienta principal [`show_widget`](/es/tools/show-widget) representa SVG o HTML autocontenido directamente desde una llamada de herramienta. El navegador y los clientes de chat nativos compatibles anuncian la capacidad `inline-widgets` del Gateway, y el documento de Canvas resultante continúa disponible cuando se vuelve a cargar el historial del chat. Las Activities de Discord proporcionan el mismo nombre de herramienta en Discord; las ejecuciones procedentes de otros canales no la reciben.

<Tabs>
  <Tab title="strict">
    Desactiva la ejecución de scripts dentro del contenido insertado alojado.
  </Tab>
  <Tab title="scripts (default)">
    Permite contenido insertado interactivo manteniendo el aislamiento del origen; suele ser suficiente para juegos y widgets de navegador autocontenidos.
  </Tab>
  <Tab title="trusted">
    Añade `allow-same-origin` además de `allow-scripts` para documentos del mismo sitio que necesitan intencionadamente privilegios más amplios.
  </Tab>
</Tabs>

```json5
{
  gateway: {
    controlUi: {
      embedSandbox: "scripts",
    },
  },
}
```

<Warning>
Utilice `trusted` solo cuando el documento insertado necesite realmente un comportamiento del mismo origen. Para la mayoría de los juegos y lienzos interactivos generados por agentes, `scripts` es la opción más segura.
</Warning>

Las URL externas absolutas de contenido insertado `http(s)` permanecen bloqueadas de forma predeterminada. Para permitir que `[embed url="https://..."]` cargue páginas de terceros, establezca `gateway.controlUi.allowExternalEmbedUrls: true`.

## Diseño de la transcripción del chat

El registro del chat utiliza un marco legible centrado y alineado con el compositor. La salida del asistente y de las herramientas permanece alineada a la izquierda, mientras que los mensajes propios permanecen alineados a la derecha dentro de ese marco. En las sesiones multiusuario (por ejemplo, un chat grupal retransmitido desde un plugin de canal), los mensajes de otros participantes identificados se muestran alineados a la izquierda con el avatar y el nombre del autor, además de un color estable por identidad, de modo que solo los mensajes del usuario que ha iniciado sesión se perciban como «míos». Cuando hay dos o más participantes identificados, las respuestas del asistente incluyen un pequeño indicador «En respuesta a nombre» que nombra al participante cuyo mensaje activó el turno. Las entradas del sistema, como la salida local de comandos con barra, se muestran como filas de aviso centradas sin avatar.

## Ancho de los mensajes del chat

Quienes usan monitores anchos pueden sustituir el ancho del registro en **Configuración → Chat →
Ancho de los mensajes**. La preferencia permanece en el almacenamiento local de ese navegador. Las formas
compatibles incluyen longitudes simples y porcentajes como `960px` o `82%`, además de
expresiones de ancho restringidas `min(...)`, `max(...)`, `clamp(...)`, `calc(...)` y
`fit-content(...)`.

## Acceso a la tailnet (recomendado)

<Tabs>
  <Tab title="Tailscale Serve integrado (preferido)">
    Mantenga el Gateway en la interfaz de bucle invertido y permita que Tailscale Serve actúe como proxy mediante HTTPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Abra `https://<magicdns>/` (o el valor `gateway.controlUi.basePath` configurado).

    De forma predeterminada, las solicitudes de Serve de la interfaz de control/WebSocket pueden autenticarse mediante los encabezados de identidad de Tailscale (`tailscale-user-login`) cuando `gateway.auth.allowTailscale` es `true`. OpenClaw verifica la identidad resolviendo la dirección `x-forwarded-for` con `tailscale whois` y comparándola con el encabezado, y solo la acepta cuando la solicitud llega mediante la interfaz de bucle invertido con los encabezados `x-forwarded-*` de Tailscale. Para las sesiones de operador de la interfaz de control con identidad de dispositivo del navegador, esta ruta de Serve verificada también omite el proceso de emparejamiento del dispositivo; los navegadores sin dispositivo y las conexiones con rol de nodo siguen las comprobaciones normales del dispositivo. Establezca `gateway.auth.allowTailscale: false` si desea exigir credenciales explícitas de secreto compartido incluso para el tráfico de Serve y, a continuación, use `gateway.auth.mode: "token"` o `"password"`.

    Para esa ruta asíncrona de identidad de Serve, los intentos de autenticación fallidos de la misma dirección IP de cliente y el mismo ámbito de autenticación se serializan antes de escribir los límites de frecuencia. Por lo tanto, los reintentos incorrectos simultáneos del mismo navegador pueden mostrar `retry later` en la segunda solicitud, en lugar de que dos discrepancias simples compitan en paralelo.

    <Warning>
    La autenticación de Serve sin token presupone que el host del Gateway es de confianza. Si puede ejecutarse código local no confiable en ese host, exija autenticación mediante token o contraseña.
    </Warning>

  </Tab>
  <Tab title="Vincular a la tailnet + token">
    ```bash
    openclaw gateway --bind tailnet --token "$(openssl rand -hex 32)"
    ```

    Abra `http://<tailscale-ip>:18789/` (o el valor `gateway.controlUi.basePath` configurado).

    Pegue el secreto compartido correspondiente en la configuración de la interfaz de usuario (se envía como `connect.params.auth.token` o `connect.params.auth.password`).

  </Tab>
</Tabs>

## HTTP no seguro

Si abre el panel mediante HTTP sin cifrar (`http://<lan-ip>` o `http://<tailscale-ip>`), el navegador se ejecuta en un **contexto no seguro** y bloquea WebCrypto. De forma predeterminada, OpenClaw **bloquea** las conexiones de la interfaz de control sin identidad de dispositivo.

La excepción compatible sin dispositivo es la autenticación correcta del operador de la interfaz de control
mediante `gateway.auth.mode: "trusted-proxy"`. No existe ningún conmutador de configuración
persistente que desactive la identidad del dispositivo.

**Solución recomendada:** use HTTPS (Tailscale Serve) o abra la interfaz de usuario localmente en `https://<magicdns>/` (Serve) o `http://127.0.0.1:18789/` (en el host del Gateway).

<AccordionGroup>
  <Accordion title="Nota sobre proxies de confianza">
    - Una autenticación correcta mediante un proxy de confianza puede admitir sesiones de **operador** de la interfaz de control sin identidad de dispositivo.
    - Esto **no** se extiende a las sesiones de la interfaz de control con rol de nodo.
    - Los proxies inversos de bucle invertido en el mismo host tampoco cumplen los requisitos de autenticación mediante proxy de confianza; consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).

  </Accordion>
</AccordionGroup>

Consulte [Tailscale](/es/gateway/tailscale) para obtener instrucciones sobre la configuración de HTTPS.

## Política de seguridad de contenido

La interfaz de control incluye una política `img-src` estricta: solo se permiten recursos del **mismo origen**, URL `data:` y URL `blob:` generadas localmente. El navegador rechaza las URL de imágenes remotas `http(s)` y las relativas al protocolo, y nunca realiza solicitudes de red para ellas.

En la práctica:

- Los avatares y las imágenes servidos mediante rutas relativas (por ejemplo, `/avatars/<id>`) siguen mostrándose, incluidas las rutas de avatares autenticadas que la interfaz de usuario obtiene y convierte en URL `blob:` locales.
- Las URL `data:image/...` insertadas directamente siguen mostrándose.
- Las URL `blob:` locales creadas por la interfaz de control siguen mostrándose.
- El Gateway obtiene de GitHub los avatares de las vistas previas de enlaces de GitHub desde su host fijo de avatares y los devuelve como URL `data:` delimitadas; el navegador del operador nunca contacta con el host remoto de avatares.
- Las URL remotas de avatares emitidas por los metadatos del canal se eliminan en los auxiliares de avatares de la interfaz de control y se sustituyen por el logotipo o distintivo integrado, de modo que un canal comprometido o malicioso no pueda forzar la obtención de imágenes remotas arbitrarias desde el navegador de un operador.

Esta protección está siempre activa y no puede configurarse.

## Autenticación de la ruta de avatares

Cuando se configura la autenticación del Gateway, el punto de conexión de avatares de la interfaz de control requiere el mismo token del Gateway que el resto de la API:

- `GET /avatar/<agentId>` devuelve la imagen del avatar únicamente a solicitantes autenticados. `GET /avatar/<agentId>?meta=1` devuelve los metadatos del avatar según la misma regla.
- Las solicitudes no autenticadas a cualquiera de las rutas se rechazan (al igual que en la ruta contigua de contenido multimedia del asistente), por lo que la ruta de avatares no puede filtrar la identidad del agente en hosts que, por lo demás, están protegidos.
- La interfaz de control reenvía el token del Gateway como encabezado de portador al obtener avatares y utiliza URL de blob autenticadas para que la imagen siga mostrándose en los paneles.

Si desactiva la autenticación del Gateway (no se recomienda en hosts compartidos), la ruta de avatares también deja de requerir autenticación, de acuerdo con el resto del Gateway.

## Autenticación de la ruta de contenido multimedia del asistente

Cuando se configura la autenticación del Gateway, las vistas previas del contenido multimedia local del asistente utilizan una ruta de dos pasos:

- `GET /__openclaw__/assistant-media?meta=1&source=<path>` requiere la autenticación normal de operador de la interfaz de control; el navegador envía el token del Gateway como encabezado de portador al comprobar la disponibilidad.
- Las respuestas de metadatos correctas incluyen un `mediaTicket` de corta duración limitado a esa ruta de origen exacta.
- Las URL de imágenes, audio, vídeo y documentos mostradas por el navegador utilizan `mediaTicket=<ticket>` en lugar del token o la contraseña activos del Gateway. El ticket caduca rápidamente y no puede autorizar otro origen.

Esto mantiene la representación del contenido multimedia compatible con los elementos multimedia nativos del navegador sin incluir credenciales reutilizables del Gateway en URL visibles de contenido multimedia.

## Enlaces de aprobación

Las notificaciones de aprobación del operador pueden incluir enlaces directos a un documento de aprobación independiente servido en el espacio de nombres reservado `${controlUiBasePath}/approve/{approvalId}` (por ejemplo, `/approve/<approvalId>` o `/openclaw/approve/<approvalId>` con una ruta base configurada). La URL permanece estable durante la vigencia de la aprobación y puede reenviarse de forma segura entre dispositivos propios: identifica la aprobación, pero nunca la autoriza.

- El Gateway reserva el espacio de nombres de un solo segmento `/approve/<approvalId>` antes que las rutas HTTP de los plugins para **todos** los métodos HTTP, por lo que una ruta de plugin nunca puede ocultar ni interceptar un documento de aprobación.
- Abrir un documento de aprobación requiere la misma autenticación del Gateway que el resto de la interfaz de control (token/contraseña, identidad de Tailscale Serve o identidad de proxy de confianza); las credenciales nunca forman parte de la URL de aprobación.
- Cuando se desactiva el servicio de la interfaz de control, las solicitudes al espacio de nombres devuelven `404` en lugar de pasar a los controladores de plugins.
- El inicio de sesión en un documento de aprobación es efímero para esa página: no sobrescribe la selección ni la configuración del Gateway guardadas por la interfaz de control completa en el mismo navegador.

El Gateway sirve archivos estáticos desde `dist/control-ui`:

```bash
pnpm ui:build
```

Base absoluta opcional (URL de recursos fijas):

```bash
OPENCLAW_CONTROL_UI_BASE_PATH=/openclaw/ pnpm ui:build
```

Desarrollo local (servidor de desarrollo independiente):

```bash
pnpm ui:dev
```

A continuación, dirija la interfaz de usuario a la URL de WebSocket del Gateway (por ejemplo, `ws://127.0.0.1:18789`).

## Página en blanco de la interfaz de control

Si el navegador carga un panel en blanco y DevTools no muestra ningún error útil, es posible que una extensión o un script de contenido inicial haya impedido la evaluación de la aplicación de módulos JavaScript. La página estática incluye un panel de recuperación HTML simple que aparece cuando `<openclaw-app>` no se registra después del inicio.

Use la acción **Volver a intentarlo** del panel después de cambiar el entorno del navegador, o recargue manualmente después de estas comprobaciones:

- Desactive las extensiones que insertan contenido en todas las páginas, especialmente las que tengan scripts de contenido `<all_urls>`.
- Pruebe con una ventana privada, un perfil de navegador limpio u otro navegador.
- Mantenga el Gateway en ejecución y compruebe la misma URL del panel después de cambiar de navegador.

## Depuración y pruebas: servidor de desarrollo + Gateway remoto

La interfaz de control está formada por archivos estáticos; el destino de WebSocket es configurable y puede diferir del origen HTTP. Esto resulta útil cuando se desea ejecutar localmente el servidor de desarrollo de Vite, pero el Gateway se ejecuta en otro lugar.

<Steps>
  <Step title="Iniciar el servidor de desarrollo de la interfaz de usuario">
    ```bash
    pnpm ui:dev
    ```
  </Step>
  <Step title="Abrir con gatewayUrl">
    ```text
    http://localhost:5173/?gatewayUrl=ws%3A%2F%2F<gateway-host>%3A18789
    ```

    Autenticación opcional de un solo uso (si es necesaria):

    ```text
    http://localhost:5173/?gatewayUrl=wss%3A%2F%2F<gateway-host>%3A18789#token=<gateway-token>
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Notas">
    - `gatewayUrl` se almacena en localStorage después de la carga y se elimina de la URL.
    - Si se pasa un punto de conexión `ws://` o `wss://` completo mediante `gatewayUrl`, codifique el valor para URL para que el navegador analice correctamente la cadena de consulta.
    - `token` debe pasarse mediante el fragmento de URL (`#token=...`) siempre que sea posible. Los fragmentos no se envían al servidor, lo que evita filtraciones mediante los registros de solicitudes y el encabezado Referer. Los parámetros de consulta heredados `?token=` todavía se importan una vez por compatibilidad, pero solo como mecanismo alternativo, y se eliminan inmediatamente después del arranque.
    - `password` se conserva únicamente en memoria.
    - Cuando se establece `gatewayUrl`, la interfaz de usuario no recurre a las credenciales de configuración ni del entorno. Proporcione `token` (o `password`) explícitamente; la ausencia de credenciales explícitas es un error.
    - Use `wss://` cuando el Gateway esté detrás de TLS (Tailscale Serve, proxy HTTPS, etc.).
    - `gatewayUrl` solo se acepta en una ventana de nivel superior (no incrustada), para evitar el secuestro de clics.
    - Las implementaciones públicas de la interfaz de control fuera de la interfaz de bucle invertido deben establecer explícitamente `gateway.controlUi.allowedOrigins` (orígenes completos). Las cargas LAN/tailnet privadas del mismo origen desde la interfaz de bucle invertido, RFC1918/enlace local, `.local`, `.ts.net` o hosts CGNAT de Tailscale se aceptan sin habilitar el mecanismo alternativo del encabezado Host.
    - El inicio del Gateway puede proporcionar orígenes locales como `http://localhost:<port>` y `http://127.0.0.1:<port>` a partir del enlace y el puerto efectivos en tiempo de ejecución, pero los orígenes de navegadores remotos siguen necesitando entradas explícitas.
    - No use `gateway.controlUi.allowedOrigins: ["*"]` salvo para pruebas locales estrictamente controladas; significa permitir cualquier origen de navegador, no «hacer coincidir cualquier host que se esté usando».
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` habilita el modo alternativo de origen mediante el encabezado Host, pero es un modo de seguridad peligroso.

  </Accordion>
</AccordionGroup>

```json5
{
  gateway: {
    controlUi: {
      allowedOrigins: ["http://localhost:5173"],
    },
  },
}
```

Detalles de configuración del acceso remoto: [Acceso remoto](/es/gateway/remote).

## Temas relacionados

- [Panel de control](/es/web/dashboard) — panel de control del Gateway
- [Comprobaciones de estado](/es/gateway/health) — supervisión del estado del Gateway
- [TUI](/es/web/tui) — interfaz de usuario de terminal
- [WebChat](/es/web/webchat) — interfaz de chat basada en navegador
