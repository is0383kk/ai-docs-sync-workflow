---
read_when:
    - Implementación o revisión de la función del panel de sesiones (tableros)
    - Cambio del alojamiento de widgets, el puente de widgets o el almacenamiento de tableros
summary: 'Paneles de sesiones: arquitectura y plan de implementación (diseño técnico, previo a la disponibilidad general)'
title: Arquitectura del panel de control
x-i18n:
    generated_at: "2026-07-26T05:35:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
Documento de diseño técnico para la funcionalidad del panel de sesiones, escrito antes y
durante la implementación. Es la fuente de referencia para el desarrollo. Cuando se
publique la funcionalidad, `/web/dashboard` se convertirá en la página orientada al usuario y esta página seguirá
siendo la referencia de arquitectura.
</Note>

## Visión

Actualmente, trabajar con un agente consiste en un flujo de texto. El panel lo convierte en un
entorno de trabajo: el agente representa widgets interactivos en tiempo real; el usuario los fija en
una superficie persistente; el chat se acopla a un lado (o se oculta) y el contenido principal es
el tablero. Se pasa de «hablar con el agente» a «operar un panel de control que el
agente ha creado» sin salir nunca de la sesión.

Principios:

- **Un tablero es una faceta de una sesión, no un objeto nuevo.** Cada sesión (hilo)
  tiene dos facetas: la transcripción y el tablero. Una sesión sin widgets fijados
  es un chat normal. Al fijar un widget, el tablero pasa a existir. Los tableros heredan la
  identidad, la propiedad del agente, el nombre, la fijación y el ciclo de vida de la sesión. No hay
  `dashboard_create`, ni registro de tableros ni un modelo de ACL independiente.
- **Paridad del agente.** Todo lo que el usuario puede hacer en un tablero, el agente puede hacerlo
  con herramientas: añadir/actualizar/eliminar widgets, organizarlos, gestionar pestañas, cambiar la
  pestaña visible y acoplar u ocultar el chat.
- **Nativo, no integrado.** El tablero consta de componentes Lit en el entorno de la interfaz de control
  (el mismo sistema de diseño que el resto de la aplicación). Solo el _contenido_ de los widgets está
  aislado en iframes. Sin barra de URL ni elementos propios del navegador.
- **Superficie reducida para el agente.** Los widgets se identifican mediante un nombre estable y se actualizan
  en el mismo lugar. El diseño es una cuadrícula fluida con compactación automática; el agente indica tamaños y
  anclajes, nunca píxeles ni coordenadas.
- **Capacidades en lugar de confianza.** El código de los widgets es HTML/JS arbitrario creado por el agente
  en un entorno aislado estricto. El acceso (datos del Gateway, acciones y red) solo existe mediante
  un manifiesto de capacidades declarado y concedido por el operador.

## Conceptos

| Concepto              | Definición                                                                                                                                                        |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sesión (hilo)         | Sesión existente del Gateway, identificada mediante el valor estable `sessionKey`. Pertenece a un agente.                                                                                        |
| Tablero               | La faceta de widgets de una sesión. Existe si y solo si la sesión tiene widgets/pestañas. Persiste tras `/new`/`/reset` (está asociado a `sessionKey`, no a la transcripción).                 |
| Pestaña               | Una página de presentación de un tablero: qué widgets contiene, su disposición y el estado de acoplamiento del chat (`left`/`right`/`bottom`/`hidden`). Los tableros comienzan con una pestaña implícita. |
| Widget                | Programa HTML/JS con nombre y aislado que pertenece a la sesión. Se identifica mediante `sessionKey` + `name`. Se actualiza en el mismo lugar por nombre.                                              |
| Manifiesto de capacidades | Declaración de acceso por widget: `data` (vinculaciones de lectura), `actions` (verbos incluidos en la lista de permitidos), `prompt` (envío a la sesión), `net` (orígenes permitidos).                      |
| Fijar (widget)        | Trasladar un widget de la transcripción al tablero de la sesión (mediante un control para el usuario o un argumento de herramienta del agente). Desfijarlo lo elimina del tablero.                                         |
| Fijar (sesión)        | Fijación existente de sesiones en la barra lateral. Una sesión fijada con tablero se abre en la faceta del tablero.                                                                      |

## Flujos de experiencia de usuario

- **Promoción:** el agente llama a `show_widget` en cualquier chat → el widget se representa en línea
  en la transcripción exactamente como hasta ahora → al pasar el cursor se muestra **Fijar en el panel** → el widget
  aparece en el tablero de la sesión. El agente puede pasar `pin: true` para hacer lo mismo.
- **Vista de tablero:** una sesión con tablero recibe un selector de faceta (Chat / Panel).
  Vista de tablero = barra de pestañas (solo cuando hay >1 pestaña) + cuadrícula fluida + panel de chat acoplado.
  El acoplamiento del chat puede redimensionarse, moverse (izquierda/derecha/parte inferior) y contraerse exactamente
  como la barra lateral. Se recuerda el estado de acoplamiento de cada pestaña.
- **Arrastre:** el usuario arrastra los widgets; la cuadrícula se compacta automáticamente (los widgets suben y los elementos adyacentes
  se redistribuyen). El redimensionamiento mediante el controlador se ajusta a incrementos de tamaño. Nadie puede realizar una colocación
  por píxeles.
- **Advertencia de restablecimiento:** `/new` / `/reset` en una sesión con tablero solicita
  confirmación en la interfaz web («se restablece el contexto, el panel permanece») y conserva
  el tablero.
- **Barra lateral:** las sesiones fijadas muestran la faceta del tablero cuando disponen de ella.
  El tablero de la sesión de inicio es el «panel del agente» predeterminado.
- **Interacciones** (tres niveles, descritos a continuación): eventos de estado silenciosos, envíos
  visibles de indicaciones y activadores de automatización.

## Niveles de interacción

1. **Eventos de estado (predeterminado).** Interacciones de la interfaz del widget que el modelo debe conocer,
   pero a las que no debe responder. `bridge.emitState({...})` añade un aviso estructurado
   de sesión (el mismo mecanismo que los avisos de actividad de grupo). No se
   inicia ningún turno del agente; el modelo ve los avisos acumulados en su siguiente ejecución.
2. **Indicaciones (conversación explícita).** `bridge.sendPrompt(text)` — requiere la activación
   del usuario; envía un mensaje visible del usuario a la sesión (el chat acoplado
   lo muestra). Está sujeto a límites de frecuencia; el usuario confirma cada envío salvo que el widget tenga
   concedida la capacidad `prompt`.
3. **Automatización.** `bridge.runAction(name, args)` — activa una acción
   declarada en el manifiesto. Conjunto inicial de verbos: `cron.trigger` (ejecutar ahora un trabajo de Cron existente) y
   `binding.refresh`. Los trabajos de Cron ya se ejecutan en sesiones de ejecución visibles y aisladas
   y pueden usar un modelo más económico: esa es la vía en la que «un modelo pequeño ejecuta el widget».
   No hay sesiones ocultas en ningún caso.

## Modelo y alojamiento de widgets

El HTML/JS del widget lo crea el agente (normalmente mediante `show_widget`), se encapsula
en el documento estándar (metadato CSP, notificador de tamaño e inicialización del puente) y se
representa en `<iframe sandbox="allow-scripts">` (nunca en `allow-same-origin`).

- **Los widgets en línea (transcripción)** mantienen el pipeline actual de documentos del lienzo:
  se escriben en el directorio de estado, los sirve el Gateway, se depuran por ámbito y no requieren
  aprobación (carecen de capacidades por definición; el usuario confirma los envíos de indicaciones).
- **Los widgets del tablero** son estado de la sesión: los bytes residen en la base de datos SQLite
  del agente propietario (`board_widgets`) y los sirve una ruta principal del Gateway
  (`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`) que lee la base de datos.
  Al fijar un widget de la transcripción se copian los bytes. Límites: 256 KB por widget,
  48 widgets por tablero.
- **Actualización en el mismo lugar:** volver a emitir un widget con el mismo `name` sustituye los
  bytes, incrementa `revision`, difunde `board.changed` y hace que las vistas activas recarguen
  solo ese iframe.
- **Inmovilización de bytes:** las capacidades concedidas se vinculan al sha256 de los bytes
  del widget. Al cambiar los bytes se conservan las concesiones `data`/`net`/`actions` solo si la nueva
  revisión declara un subconjunto del manifiesto concedido; un manifiesto ampliado
  vuelve a solicitar confirmación al operador.

### Los widgets alojan contenido; las aplicaciones MCP son un tipo de contenido

El **widget es el elemento básico de OpenClaw**: la celda del tablero con nombre, fijada, dimensionada,
propiedad de la sesión y con un registro de concesión. Lo que se representa dentro es un
tipo de contenido:

- `html` — creado por el agente mediante `show_widget`, con los bytes en el almacenamiento del tablero.
- `mcp-app` — una vista de aplicación MCP de terceros (recurso `ui://` de un servidor
  configurado) alojada dentro de la celda del widget.

Las aplicaciones MCP no definen el modelo de widgets; los widgets adquirieron la capacidad de
alojarlas. La identidad, la colocación, la fijación, las concesiones y la API para autores siguen
siendo propiedad de OpenClaw, por lo que el código de `show_widget` sigue siendo tan breve como hasta ahora y nunca
necesita saber que existe la especificación de aplicaciones MCP.

Infraestructura compartida subyacente (aquí es donde se materializa la simplificación):

- **Un único host aislado.** Los widgets `html` se representan mediante el mismo pipeline reforzado
  con el que se publicaron las aplicaciones MCP (doble iframe en el origen dedicado al aislamiento,
  CSP declarada por widget, decodificada con cierre seguro en caso de error), en lugar de un segundo
  host de iframe específico. El proxy recibe HTML por valor, por lo que el contenido local es
  el caso natural.
- **Un único modelo de autorización.** El acceso de un widget es una lista de permitidos concedida,
  cualquiera que sea su tipo: para los widgets `html`, herramientas del host; para los widgets `mcp-app`,
  las herramientas del servidor visibles para la aplicación (mediante el mecanismo `allowedAppToolNames`
  existente, que pasa a ser duradero por widget en lugar de por ejecución de creación).
- **Herramientas del host para los widgets `html`** (expuestas mediante el puente del widget y comprobadas
  con respecto a la concesión):
  - `openclaw.prompt.send` — nivel 2; se enruta mediante el redactor visible,
    con confirmación del usuario salvo que se haya concedido
  - `openclaw.state.emit` — avisos de sesión de nivel 1 (agrupados y con límite de tamaño)
  - `openclaw.data.read` — vinculaciones parametrizadas de solo lectura (conjunto existente
    de RPC de lectura incluidos en la lista de permitidos), resueltas en el Gateway
  - `openclaw.cron.trigger` — automatización de nivel 3
- **`net` = CSP.** El acceso a la red usa la declaración CSP por widget
  ya publicada (orígenes `connect-src`): el widget meteorológico que se actualiza automáticamente
  obtiene su API directamente desde el entorno aislado, sin intervención del Gateway.
- **Concesiones.** Un widget que no declara nada se representa inmediatamente (aislado,
  `default-src 'none'`, con confirmación individual de los envíos de indicaciones), con la misma confianza que
  los widgets en línea de los chats actuales. Las herramientas/orígenes declarados colocan el widget en
  `pending` en el tablero: una tarjeta de marcador de posición los enumera de forma comprensible con
  las opciones **Permitir**/**Rechazar** de un solo toque. Las concesiones se asignan por nombre de widget; para los widgets `html`
  están inmovilizadas por bytes (sha256), y los bytes modificados solo conservan la concesión si la
  declaración se ha reducido.
- **Capa de compatibilidad para autores.** El contenedor del documento inyecta `window.openclaw.prompt`,
  `window.openclaw.state`, `window.openclaw.data` y `window.openclaw.cron`
  como API estable para autores. Las llamadas del panel comparten un único canal de solicitudes
  vinculado al ticket de la vista; la notificación de tamaño y los tokens del tema siguen siendo notificaciones
  independientes del host.

### Declaraciones de capacidades de los plugins

Los plugins habilitados pueden ampliar el host de widgets mediante `dashboard.dataBindings`
y `dashboard.actionVerbs` en `openclaw.plugin.json`. Los identificadores locales del plugin se convierten
en nombres de concesión con el identificador del plugin como prefijo, como `workboard.cards.list` y
`workboard.dispatch`; `%` y `.` se escapan en el segmento del identificador del plugin para que una
división diferente entre plugin e identificador local no pueda heredar la misma concesión persistente. Durante
el registro del plugin, OpenClaw verifica que cada vinculación apunte a un RPC
registrado por el mismo plugin con `operator.read` y que cada acción apunte a uno
con `operator.write`; las declaraciones no válidas hacen que falle la carga del plugin. El registro validado
solo se reconstruye cuando cambia el ciclo de vida de los plugins, mientras que las concesiones de los widgets
siguen siendo específicas de cada widget y vinculadas a los bytes y la revisión.

### Limitación residual modelada: canales de datos WebRTC

La CSP del entorno aislado emite la directiva propuesta `webrtc 'block'`, pero
[el conjunto actual de directivas CSP de Chromium](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)
no la implementa. Por tanto, los widgets programables pueden usar canales de datos
WebRTC para la salida de datos en la versión actual de Chromium. Esta misma limitación residual ya está presente en
los widgets de chat en línea y en el host de aplicaciones MCP en `main`.

**Compensación aceptada:** OpenClaw no condiciona los widgets programables a este
riesgo residual. El contenido de los widgets obtiene acceso a datos confidenciales de OpenClaw únicamente mediante
una capacidad `data:read` concedida por el operador e inmutable a nivel de bytes, y la política de permisos
del entorno aislado bloquea el acceso a la cámara y al micrófono. Una protección de la API del DOM es
una defensa en profundidad basada en el mejor esfuerzo, no un límite de seguridad, y corresponde
a un refuerzo posterior.

### Visualización de la transcripción: una tarjeta de widget

La visualización en línea se unifica en torno a la primitiva de widget. Cuando el resultado de una herramienta contiene una interfaz de usuario —
la salida de `show_widget` o el resultado de una herramienta MCP con un recurso de aplicación—, el sistema
materializa un **widget efímero con nombre automático** (limitado a la sesión y depurado) y
la transcripción representa una única tarjeta de widget que se despacha según el tipo de contenido.
La visualización automática de aplicaciones MCP se mantiene exactamente como espera la especificación (sin trabajo adicional del modelo);
simplemente _es_ un widget internamente. Esto elimina los casos especiales paralelos de `mcpApp`
en la representación del chat (condicionamiento por superficie y desduplicación separada), proporciona a todas
las interfaces en línea el mismo control para fijarlas y convierte el registro de widgets en la ruta principal
para volver a abrirlas (la reconstrucción mediante el análisis de la transcripción se mantiene como alternativa para el historial
que nunca se haya fijado). El host independiente de solo lectura con tickets se solapa con los tableros como una
superficie persistente para volver a abrir elementos; es un candidato a consolidación que se evaluará en T6, no
algo que se dé por supuesto.

Composición: v1 usa adyacencia en cuadrícula (widget de interfaz del agente junto a un widget de aplicación en
una pestaña). v2 añade **ranuras de aplicación gestionadas por el host**: el HTML del widget del agente declara una
región de ranura y el host compone la vista real de la aplicación como un entorno aislado hermano.
La aplicación nunca se representa dentro del iframe del agente: el anidamiento rompería la
identidad del puente y permitiría superponer o secuestrar clics de la interfaz de la aplicación autorizada, por lo que la ranura es un
contrato de diseño, no una incrustación.

### Widgets proporcionados por el servidor (aplicaciones MCP fijadas)

Con el host unificado, fijar una aplicación MCP de terceros consiste simplemente en un widget cuyo
contenido se obtiene del servidor en lugar de almacenarse: `board_widgets` conserva el
descriptor (`serverName`, `toolName`, `uiResourceUri`, `toolCallId` + `sessionKey` de origen) en lugar de los bytes HTML, y el tablero vuelve a emitir el
arrendamiento de la vista después del TTL de 10 minutos del turno de chat (volviendo a obtener el recurso `ui://`
cuando queda obsoleto). Las vistas de aplicaciones MCP en línea del chat reciben el mismo control **Fijar al panel**
que los widgets del agente. Actualmente, las vistas reabiertas son de solo lectura por diseño;
las aplicaciones fijadas que deban seguir siendo interactivas reciben una concesión duradera sobre las herramientas
del servidor visibles para la aplicación (una lista explícita de elementos permitidos que se muestra al operador al fijarla), desacoplada
de la ejecución que la emitió. Los elementos fijados sin concesión permanecen en modo de solo lectura, aunque siguen siendo útiles para paneles
de visualización. v1 fija elementos en el tablero de la sesión de origen; fijarlos entre sesiones
requiere un intermediario de arrendamientos y tendrá que esperar. Se debe coordinar con el pull request abierto #109807 (enrutamiento del compositor `ui/message`,
propagación del tema y el tamaño).

### Integración con WorkBoard

El programa de integración con WorkBoard mantiene las tarjetas y los tableros bajo la propiedad del plugin, a la vez que vuelve a enlazar las tarjetas despachadas con los tableros de sus sesiones mediante los elementos existentes `sessionKey` y `runId`, expone los canales de WorkBoard y el despacho mediante enlaces y acciones declarados por el plugin, y compone esos resultados con los tipos de widget existentes `html` y `mcp-app` en lugar de introducir un tipo de widget específico de WorkBoard.

## Diseño: cuadrícula fluida

12 columnas, altura de fila fija, **compactación automática** (gravedad hacia arriba, desplazamiento lateral al
arrastrar; semántica de gridstack, implementada de forma nativa; las operaciones matemáticas de la cuadrícula permanecen puras y
sin DOM). Estado de diseño de los widgets por pestaña: `{ name, w (1-12), h (rows) }` más
el orden. Vocabulario del agente:

- `size`: `sm` (3×3) · `md` (6×4) · `lg` (8×6) · `xl` (12×8) · `full`
  (pestaña de un solo widget)
- `after: <widgetName>` ancla de ordenación opcional; si se omite, se añade al final
- La persona usuaria arrastra y cambia el tamaño libremente; el mismo modelo de orden y tamaño admite el recorrido de ida y vuelta.

## Modelo de datos (base de datos por agente)

Nuevas tablas en `agents/<agentId>/agent/openclaw-agent.sqlite`
(**requiere incrementar la versión del esquema de la base de datos del agente; se necesita la aprobación del operador
antes de incorporarlo**):

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- identificador legible
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- nombre estable del widget
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- código fuente del documento envuelto
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- orden dentro de la pestaña (entrada de compactación automática)
  manifest     TEXT NOT NULL DEFAULT '{}',  -- JSON del manifiesto de capacidades
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- concesión inmutable a nivel de bytes
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

La existencia del tablero se determina por la presencia de cualquier fila para `sessionKey`. Al eliminar una sesión, se eliminan las filas de su
tablero. `/new`/`/reset` no las modifica.

## Superficie del protocolo

RPC (tabla de métodos del núcleo, esquemas typebox en `gateway-protocol`):

- `board.get { sessionKey }` → pestañas + metadatos de widgets (sin bytes) — `operator.read`
- `board.update { sessionKey, ops[] }` — CRUD/reordenación de pestañas, traslado/cambio de tamaño/
  eliminación/desfijación de widgets, estado del acoplamiento, enfoque de pestaña — `operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }` —
  `operator.write` (ruta de herramientas del agente y ruta para fijar)
- `board.widget.grant { sessionKey, name, decision }` — `operator.approvals`
- `board.event { ticket, payload }` — ingesta de eventos de estado de nivel 1 vinculada a tickets;
  se conserva la forma heredada `{ sessionKey, widget, payload }` del host de confianza —
  `operator.write`
- `board.prompt.authorize { ticket }` — devuelve si el envío de una instrucción visible
  todavía requiere confirmación con cada clic — `operator.read`
- `board.data.read { ticket, bindingId, params? }` — resolución, del lado del Gateway, de enlaces de lectura de
  núcleo o plugins activos incluidos en la lista de permitidos — `operator.read`
- `board.action { ticket, action, ... }` — despacho de automatización con concesión exacta
  mediante la ruta existente de ejecución inmediata de cron o el verbo de acción validado de un plugin
  activo — `operator.write`

Eventos (en `EVENT_SCOPE_GUARDS`, ámbito de lectura):

- `board.changed { sessionKey, revision, widget? }` — el estado persistente ha cambiado;
  la interfaz vuelve a obtener los datos (y recarga un iframe cuando está presente `widget`).
- `board.command { sessionKey, command }` — control transitorio de la interfaz (el agente cambia
  la pestaña visible o alterna el acoplamiento del chat) — el patrón `ui.command`.

Los bytes de los widgets se sirven mediante la superficie HTTP autenticada, no mediante el socket.

## Herramientas del agente

Tres herramientas en total (del núcleo, siempre registradas; la representación está condicionada por la
capacidad del cliente `inline-widgets`, como actualmente):

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }` — crea/actualiza por nombre; `pin` lo coloca en el tablero.
  Sin `name`/`pin`, se comporta exactamente como ahora (en línea y efímero).
- `dashboard { action, ... }` — verbos de gestión del tablero: `read`, `tab_create`,
  `tab_update`, `tab_delete`, `tabs_reorder`, `widget_move`, `widget_remove`,
  `unpin`, `focus_tab`, `set_chat_dock`.
- Las herramientas existentes `cron` cubren el nivel de automatización; no se necesita ninguna herramienta nueva.

Las descripciones de las herramientas enseñan el vocabulario de tamaños y anclas, así como el modelo de niveles. Se
informa al agente sobre los eventos de nivel 1 de la persona usuaria mediante avisos de sesión, por ejemplo,
`[dashboard] user clicked "Refresh" on widget weather (tab main)`.

## Qué sustituye esto

- **Se elimina `extensions/workspaces`.** Era experimental, `enabledByDefault:
false`, y nunca apareció en una versión estable (se introdujo por primera vez en las betas de 2026.7.2). No hay
  migración; una regla de doctor elimina `<stateDir>/workspaces/` obsoleto si está presente.
  Ideas aprovechadas: operaciones matemáticas puras de cuadrícula, modelo de seguridad del puente (inicialización del puerto,
  condicionamiento de enlaces, límites de frecuencia), aprobación inmutable a nivel de bytes.
- **El alojamiento de widgets se traslada de `extensions/canvas` al núcleo.** El almacén de documentos del lienzo,
  el envoltorio de documentos, el servicio HTTP y la herramienta `show_widget` pasan a formar parte del núcleo
  (`src/canvas/`); el plugin conserva la herramienta de control node-canvas (`canvas`) y
  A2UI. El anuncio `pluginSurfaceUrls["canvas"]` y
  las rutas `/__openclaw__/canvas` son contratos publicados de clientes nativos y permanecen
  estables. Las sesiones de Discord conservan la variante `show_widget` propiedad de Discord.

## Objetivos excluidos (este programa)

- Uso compartido de tableros entre varias personas/ACL (futuro; llegará mediante el uso compartido de sesiones).
- Representación nativa de tableros en macOS/iOS (estará disponible donde incorporen la
  interfaz de control; la ruta de widgets en línea no cambia).
- Widgets de datos integrados (tarjetas de sesiones/uso/cron): el puente de capacidades más
  los widgets creados por agentes cubren v1; más adelante puede añadirse un registro de tipos integrados.

## Plan de implementación

Árboles de trabajo independientes, creados con Codex, con revisión e incorporación secuenciales. Incorporar y luego corregir.

| #   | Rama                                 | Alcance                                                                                                                                                                            | Depende de                        |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | Eliminar el plugin de espacios de trabajo + interfaz + documentación + claves de i18n; regla de limpieza de doctor                                                                 | —                                |
| T2  | `claude/dashboard-canvas-core`       | Promover el alojamiento de widgets + `show_widget` al núcleo; el plugin de lienzo conserva la herramienta de Node; ningún cambio de comportamiento                                 | —                                |
| T3  | `claude/dashboard-domain`            | Tablas de la base de datos del agente (incremento de esquema), RPC `board.*` + eventos, herramienta `dashboard`, argumentos de fijación/nombre/manifiesto `show_widget`, avisos de nivel 1, el restablecimiento conserva el tablero | T2                               |
| T4  | `claude/dashboard-ui`                | Vista del tablero + barra de pestañas + cuadrícula fluida con compactación automática + acoplamiento del chat (izquierda/derecha/abajo/oculto) + control de fijación en la transcripción + vista del tablero en la barra lateral + confirmación de restablecimiento | T3 (primero simulaciones mediante accesorios de desarrollo) |
| T5  | `claude/dashboard-capabilities`      | Almacén/interfaz de concesiones + inmutabilidad a nivel de bytes; trasladar los widgets `html` al host compartido del entorno aislado; herramientas del host (`openclaw.prompt.send/state.emit/data.read/cron.trigger`); CSP `net`; capa de compatibilidad de creación | T3, T4                           |
| T7  | `claude/dashboard-mcp-apps`          | Tipo de contenido `mcp-app`: control para fijar en las vistas de aplicaciones en línea, almacenamiento de descriptores, nueva emisión/actualización de arrendamientos, concesiones duraderas de herramientas del servidor (reutiliza el host publicado de aplicaciones MCP) | T3, T4                           |
| T6  | perfeccionamiento                    | E2E en vivo en un Gateway temporal (claves reales), capturas de pantalla, correcciones, reescritura de `/web/dashboard` centrada en la persona usuaria, revisión de la activación predeterminada | todos                            |

Validación según las reglas del repositorio: vitest específico en local, comprobaciones completas en
Crabbox/Testbox, `$autoreview` antes de cada incorporación y prueba en vivo para T6.
