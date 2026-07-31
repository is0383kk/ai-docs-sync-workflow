---
read_when:
    - Añadir funciones que amplían el acceso o la automatización
summary: Consideraciones de seguridad y modelo de amenazas para ejecutar un gateway de IA con acceso al shell
title: Seguridad
x-i18n:
    generated_at: "2026-07-26T04:38:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8cdf1b1455ecb35a3cf5b9ab968a55c89b7b7c283231b99d4d740bb75fa11700
    source_path: gateway/security/index.md
    workflow: 16
---

<Warning>
  **Modelo de confianza del asistente personal.** Esta guía presupone un único
  límite de operador de confianza por Gateway (modelo de usuario único y asistente personal).
  OpenClaw **no** constituye un límite de seguridad multiinquilino hostil para varios
  usuarios adversarios que compartan un agente o Gateway. Para operaciones con distintos niveles de confianza o
  usuarios adversarios, separe los límites de confianza: Gateway +
  credenciales independientes y, preferiblemente, usuarios del sistema operativo o hosts independientes.
</Warning>

## Alcance: modelo de seguridad del asistente personal

- Compatible: un límite de usuario/confianza por Gateway (preferiblemente, un usuario del sistema operativo/host/VPS por límite).
- No compatible: un Gateway/agente compartido que utilicen usuarios que no confían entre sí o usuarios adversarios.
- El aislamiento de usuarios adversarios requiere Gateways independientes (y, preferiblemente, usuarios del sistema operativo/hosts independientes).
- Si varios usuarios que no son de confianza pueden enviar mensajes a un agente con herramientas habilitadas, comparten la autoridad delegada de las herramientas de ese agente.
- Si alguien puede modificar el estado o la configuración del host del Gateway (`~/.openclaw`, incluido `openclaw.json`), debe considerarse un operador de confianza.
- Dentro de un Gateway, el acceso autenticado del operador es un rol de confianza del plano de control, no un rol de inquilino por usuario.
- `sessionKey` (identificadores y etiquetas de sesión) es un selector de enrutamiento, no un token de autorización.

¿Se alojan varios usuarios u organizaciones? Ejecute una celda de Gateway aislada por inquilino en lugar de compartir un Gateway. Consulte [Alojamiento multiinquilino](/es/gateway/multi-tenant-hosting).

Antes de cambiar el acceso remoto, la política de mensajes directos, el proxy inverso o la exposición pública, siga el [procedimiento de exposición del Gateway](/es/gateway/security/exposure-runbook) como lista de comprobación previa y de reversión.

## `openclaw security audit`

Ejecute lo siguiente después de cualquier cambio de configuración o antes de exponer superficies de red:

```bash
openclaw security audit
openclaw security audit --deep    # intenta realizar una comprobación activa del Gateway
openclaw security audit --fix     # aplica correcciones seguras
openclaw security audit --json
```

`--fix` tiene un alcance deliberadamente limitado: cambia las políticas de grupos abiertos a listas de permitidos, restaura `logging.redactSensitive: "tools"`, restringe los permisos de los archivos de estado, configuración e inclusión (archivos `600`, directorios `700`) y, en Windows, utiliza restablecimientos de ACL en lugar de `chmod` de POSIX.

### Qué comprueba la auditoría (a grandes rasgos)

- **Acceso entrante**: políticas de mensajes directos y grupos, listas de permitidos: ¿pueden personas desconocidas activar el bot?
- **Radio de impacto de las herramientas**: herramientas con privilegios elevados y salas abiertas: ¿podría una inyección de instrucciones convertirse en acciones de shell, archivos o red?
- **Desviación del sistema de archivos de ejecución**: herramientas que modifican el sistema de archivos denegadas mientras `exec`/`process` siguen disponibles sin restricciones de entorno aislado.
- **Desviación de las aprobaciones de ejecución**: `security="full"`, `autoAllowSkills`, listas de intérpretes permitidos sin `strictInlineEval`. `security="full"` por sí solo es una advertencia general sobre la postura, no una prueba de un error: es el valor predeterminado elegido para las configuraciones de asistentes personales de confianza; restrínjalo únicamente cuando el modelo de amenazas requiera mecanismos de protección mediante aprobaciones o listas de permitidos.
- **Exposición de red**: enlace/autenticación del Gateway, Serve/Funnel de Tailscale, tokens de autenticación débiles o cortos.
- **Exposición del control del navegador**: nodos remotos, puertos de retransmisión y puntos de conexión CDP remotos.
- **Higiene del disco local**: permisos, enlaces simbólicos, inclusiones de configuración y rutas de carpetas sincronizadas.
- **Plugins**: carga sin una lista de permitidos explícita.
- **Desviación de las políticas**: ajustes de Docker del entorno aislado configurados pero modo de entorno aislado desactivado; entradas `gateway.nodes.commands.deny` que parecen efectivas, pero solo coinciden con identificadores exactos de comandos (por ejemplo, `system.run`), no con el texto de shell dentro de la carga útil; entradas `gateway.nodes.commands.allow` peligrosas; `tools.profile="minimal"` global anulado por agente; herramientas propiedad de plugins accesibles mediante una política permisiva.
- **Desviación de las expectativas del entorno de ejecución**: suponer que la ejecución implícita todavía significa `sandbox` cuando `tools.exec.host` ahora utiliza `auto` de forma predeterminada, o establecer `tools.exec.host="sandbox"` mientras el modo de entorno aislado está desactivado.
- **Higiene de los modelos**: advierte sobre modelos heredados configurados (advertencia leve, no bloqueo estricto).

Cada hallazgo tiene un `checkId` estructurado (por ejemplo, `gateway.bind_no_auth`, `tools.exec.security_full_configured`). Prefijos: `fs.*` (permisos), `gateway.*` (enlace/autenticación/Tailscale/Control UI/proxy de confianza), `hooks.*`/`browser.*`/`sandbox.*`/`tools.exec.*` (refuerzo por superficie), `plugins.*`/`skills.*` (cadena de suministro), `security.exposure.*` (política de acceso × radio de impacto de las herramientas). Catálogo completo con gravedad y compatibilidad con correcciones automáticas: [Comprobaciones de la auditoría de seguridad](/es/gateway/security/audit-checks). Consulte también [Verificación formal](/es/security/formal-verification).

### Orden de prioridad al clasificar los hallazgos

1. Cualquier elemento «abierto» con herramientas habilitadas: restrinja primero los mensajes directos y los grupos (emparejamiento/listas de permitidos) y, después, endurezca la política de herramientas y el entorno aislado.
2. Exposición a redes públicas (enlace LAN, Funnel, ausencia de autenticación): corríjala de inmediato.
3. Exposición remota del control del navegador: trátela como acceso de operador (solo en la red privada de Tailscale, empareje los nodos deliberadamente y evite la exposición pública).
4. Permisos: el estado, la configuración, las credenciales y la autenticación no deben ser legibles por el grupo ni por todos los usuarios.
5. Plugins: cargue únicamente aquellos en los que confíe explícitamente.
6. Elección del modelo: para cualquier bot con herramientas, prefiera modelos modernos y reforzados frente a instrucciones maliciosas.

## Configuración base reforzada en 60 segundos

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
  channels: {
    whatsapp: { dmPolicy: "pairing", groups: { "*": { requireMention: true } } },
  },
}
```

Mantiene el Gateway accesible solo localmente, aísla los mensajes directos y deshabilita de forma predeterminada las herramientas del plano de control y del entorno de ejecución. A partir de esta configuración, vuelva a habilitar selectivamente las herramientas para cada agente de confianza.

Configuración base integrada para los turnos de agentes controlados mediante chat: los remitentes que no sean propietarios no pueden utilizar las herramientas `cron` ni `gateway`, independientemente de la configuración.

### Controles limitados al solicitante y contexto de las instrucciones

`tools.toolsBySender`, la propiedad del remitente y los inventarios de herramientas exclusivas del propietario se evalúan en relación con el solicitante que originó el turno actual. No autentican ni depuran otros contenidos de las instrucciones del modelo, incluidos el texto citado, el historial previo de salas compartidas, el contenido reenviado, el contenido obtenido, los archivos adjuntos, los resultados de herramientas u otras entradas de las instrucciones. Por tanto, el contenido de otra persona puede influir en un turno iniciado por el propietario cuando se incluye en el contexto de ese turno.

Trate estos controles como defensa en profundidad que reduce la capacidad directa de un solicitante, no como aislamiento multiusuario hostil. Utilice `contextVisibility` para filtrar el contexto compatible proporcionado por el canal, restrinja las herramientas y aísle el agente; además, utilice Gateways independientes y, preferiblemente, usuarios del sistema operativo o hosts independientes cuando los participantes sean mutuamente adversarios.

## Matriz de límites de confianza

Modelo rápido para clasificar informes de riesgos:

| Límite o control                                          | Qué significa                                                | Interpretación errónea habitual                                                   |
| --------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| `gateway.auth` (token/contraseña/proxy de confianza/autenticación de dispositivo) | Autentica a quienes llaman a las API del Gateway             | «Necesita firmas por mensaje en cada trama para ser seguro»                       |
| `sessionKey`                                              | Clave de enrutamiento para seleccionar el contexto o la sesión | «La clave de sesión es un límite de autenticación del usuario»                    |
| Mecanismos de protección de instrucciones/contenido       | Reducen el riesgo de abuso del modelo                        | «La inyección de instrucciones por sí sola demuestra una omisión de autenticación» |
| `canvas.eval` / evaluación del navegador                  | Capacidad intencional del operador cuando está habilitada    | «Cualquier primitiva de evaluación de JS es automáticamente una vulnerabilidad en este modelo de confianza» |
| Shell `!` de la TUI local                              | Ejecución local iniciada explícitamente por el operador      | «Un comando de conveniencia del shell local es una inyección remota»              |
| Emparejamiento y comandos de nodos                        | Ejecución remota de nivel de operador en dispositivos emparejados | «El control de dispositivos remotos debe tratarse de forma predeterminada como acceso de usuarios que no son de confianza» |
| `gateway.nodes.pairing.autoApproveCidrs`                  | Política opcional de incorporación de nodos de redes de confianza | «Una lista de permitidos deshabilitada de forma predeterminada es automáticamente una vulnerabilidad de emparejamiento» |
| `gateway.nodes.pairing.sshVerify`                         | Incorporación de nodos verificada mediante claves a través del SSH del operador | «La aprobación automática activada de forma predeterminada es automáticamente una vulnerabilidad de emparejamiento» |

## No son vulnerabilidades por diseño

<Accordion title="Hallazgos habituales cerrados sin intervención">

- Cadenas basadas únicamente en la inyección de instrucciones sin eludir políticas, autenticación ni entornos aislados.
- Afirmaciones que presuponen una operación multiinquilino hostil en un único host o configuración compartidos.
- Acceso normal del operador mediante rutas de lectura (por ejemplo, `sessions.list` / `sessions.preview` / `chat.history`) clasificado como IDOR en una configuración con Gateway compartido.
- Hallazgos de implementaciones accesibles solo mediante localhost (por ejemplo, ausencia de HSTS en un Gateway accesible únicamente mediante bucle local).
- Hallazgos relacionados con firmas de Webhook entrantes de Discord para rutas entrantes que no existen en este repositorio.
- Metadatos de emparejamiento de nodos tratados como una segunda capa oculta de aprobación por comando para `system.run`; el límite de ejecución real es la política global de comandos de nodos del Gateway junto con las propias aprobaciones de ejecución del nodo.
- `gateway.nodes.pairing.sshVerify` tratado como una vulnerabilidad porque está habilitado de forma predeterminada. Nunca aprueba basándose únicamente en la ubicación de red o en la accesibilidad mediante SSH: el Gateway vuelve a leer la identidad del dispositivo mediante SSH (BatchMode, claves de host estrictas) y solo aprueba cuando la clave del dispositivo coincide exactamente con la solicitud pendiente, lo que requiere que el par de claves de conexión ya se encuentre en la cuenta del operador en un host que este controla. Las comprobaciones se limitan a direcciones de origen privadas/CGNAT, comparten el umbral de elegibilidad de CIDR de confianza (solo `role: node` reciente y sin ámbitos) y `sshVerify: false` desactiva la función.
- `gateway.nodes.pairing.autoApproveCidrs` tratado como una vulnerabilidad por sí solo. Está deshabilitado de forma predeterminada, requiere entradas CIDR/IP explícitas, solo se aplica al primer emparejamiento de `role: node` sin ámbitos solicitados y nunca aprueba automáticamente operadores, navegadores, Control UI, WebChat, ampliaciones de roles o ámbitos, cambios de metadatos o claves públicas ni rutas de encabezados de proxy de confianza mediante bucle local en el mismo host (incluso cuando la autenticación del proxy de confianza mediante bucle local está habilitada).
- Hallazgos de «ausencia de autorización por usuario» que tratan `sessionKey` como un token de autenticación.

</Accordion>

## Confianza del Gateway y los nodos

Trate el Gateway y el nodo como un único dominio de confianza del operador con funciones diferentes:

- **Gateway**: plano de control y superficie de políticas (`gateway.auth`, política de herramientas, enrutamiento).
- **Node**: superficie de ejecución remota emparejada con ese Gateway (comandos, acciones del dispositivo, capacidades locales del host).
- Un solicitante autenticado en el Gateway es de confianza dentro del ámbito del Gateway; tras el emparejamiento, las acciones del nodo son acciones de operador de confianza en ese nodo. Consulte [Ámbitos del operador](/es/gateway/operator-scopes).
- Los clientes directos del backend de bucle invertido autenticados con el token o la contraseña compartidos del gateway pueden realizar RPC internas del plano de control sin presentar una identidad de dispositivo de usuario. Esto no omite el emparejamiento remoto ni el del navegador: los clientes de red, los clientes de nodo, los clientes con token de dispositivo y las identidades de dispositivo explícitas siguen sujetos al emparejamiento y a la aplicación de las ampliaciones de ámbito.
- Las aprobaciones de ejecución (lista de permitidos + consulta) son mecanismos de protección para la intención del operador, no aislamiento hostil entre varios inquilinos. Vinculan el contexto exacto de la solicitud y, con el mejor esfuerzo, los operandos directos de archivos locales; no modelan semánticamente todas las rutas de carga del entorno de ejecución o del intérprete. Utilice aislamiento de procesos y del host para establecer límites sólidos.
- Valor predeterminado para un único operador de confianza: la ejecución en el host mediante `gateway`/`node` se permite sin solicitudes de aprobación (`security="full"`, `ask="off"`). Esto forma parte intencional de la experiencia de usuario y no constituye por sí mismo una vulnerabilidad.

Para aislar a usuarios hostiles, separe los límites de confianza por usuario del sistema operativo o por host y ejecute gateways independientes.

## Modelo de amenazas

El asistente de IA puede ejecutar comandos de shell arbitrarios, leer y escribir archivos, acceder a servicios de red y enviar mensajes a cualquier persona (si tiene acceso a los canales). Quienes le envíen mensajes pueden intentar engañarlo para que realice acciones perjudiciales, obtener acceso a los datos mediante ingeniería social o sondear detalles de la infraestructura.

La mayoría de los fallos en este contexto no son vulnerabilidades exóticas, sino casos en los que «alguien envió un mensaje al bot y este hizo lo que se le pidió». La postura de OpenClaw, en orden, es la siguiente:

1. **Primero, la identidad**: decida quién puede comunicarse con el bot (emparejamiento de mensajes directos, listas de permitidos o apertura explícita).
2. **Después, el ámbito**: decida dónde puede actuar el bot (listas de grupos permitidos y requisito de mención, herramientas, aislamiento y permisos de dispositivos).
3. **Por último, el modelo**: dé por sentado que el modelo puede ser manipulado; diseñe el sistema para que la manipulación tenga un radio de impacto limitado.

## Acceso mediante mensajes directos: emparejamiento, lista de permitidos, abierto y deshabilitado

Todos los canales compatibles con mensajes directos admiten `dmPolicy` (o `*.dm.policy`), que filtra los mensajes directos entrantes antes de procesarlos:

| Política      | Comportamiento                                                                                                                                                                                                             |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pairing`   | Valor predeterminado. Los remitentes desconocidos reciben un código de emparejamiento; el bot los ignora hasta que se aprueben. Los códigos caducan al cabo de 1 hora; los mensajes directos repetidos no vuelven a enviar un código hasta que se crea una nueva solicitud. El límite de solicitudes pendientes es de 3 por canal. |
| `allowlist` | Los remitentes desconocidos se bloquean sin iniciar un protocolo de emparejamiento.                                                                                                                                                                       |
| `open`      | Cualquiera puede enviar mensajes directos (público). Requiere que la lista de permitidos del canal incluya `"*"` (aceptación explícita).                                                                                                                           |
| `disabled`  | Los mensajes directos entrantes se ignoran por completo.                                                                                                                                                                                        |

```bash
openclaw pairing list <channel>
openclaw pairing approve <channel> <code>
```

Detalles y archivos en disco: [Emparejamiento](/es/channels/pairing)

Trate `dmPolicy="open"` y `groupPolicy="open"` como opciones de último recurso; prefiera el emparejamiento y las listas de permitidos, salvo que confíe plenamente en todos los miembros de la sala.

### Listas de permitidos (dos niveles)

- **Lista de permitidos para mensajes directos** (`allowFrom` / `channels.discord.allowFrom` / `channels.slack.allowFrom`; heredado: `channels.discord.dm.allowFrom`, `channels.slack.dm.allowFrom`): determina quién puede enviar mensajes directos al bot. Cuando se usa `dmPolicy="pairing"`, las aprobaciones se escriben en `~/.openclaw/credentials/<channel>-allowFrom.json` (cuenta predeterminada) o `<channel>-<accountId>-allowFrom.json` (cuentas no predeterminadas) y se combinan con las listas de permitidos de la configuración.
- **Lista de grupos permitidos** (específica del canal): determina qué grupos, canales o servidores acepta el bot.
  - `channels.whatsapp.groups`, `channels.telegram.groups`, `channels.imessage.groups`: valores predeterminados por grupo, como `requireMention`; cuando se definen, también actúan como lista de grupos permitidos (incluya `"*"` para conservar el comportamiento de permitir todo). Personalice los activadores por mención mediante `agents.entries.*.groupChat.mentionPatterns` (por ejemplo, `["@openclaw", "@mybot"]`) para que `requireMention` aplique el requisito según los nombres propios del bot.
  - `groupPolicy="allowlist"` + `groupAllowFrom`: restringen quién puede activar el bot dentro de una sesión de grupo (WhatsApp/Telegram/Signal/iMessage/Microsoft Teams).
  - `channels.discord.guilds` / `channels.slack.channels`: listas de permitidos y valores predeterminados de mención para cada superficie.
  - Orden de comprobación: primero `groupPolicy`/las listas de grupos permitidos y, después, la activación mediante mención o respuesta. Responder a un mensaje del bot (mención implícita) **no** omite `groupAllowFrom`.

Detalles: [Configuración](/es/gateway/configuration) y [Grupos](/es/channels/groups)

### Aislamiento de sesiones de mensajes directos (modo multiusuario)

De forma predeterminada, OpenClaw dirige todos los mensajes directos a la sesión principal para mantener la continuidad entre dispositivos. Si varias personas pueden enviar mensajes directos al bot (mensajes directos abiertos o una lista de permitidos con varias personas), aísle las sesiones de mensajes directos:

```json5
{ session: { dmScope: "per-channel-peer" } }
```

Valores de `session.dmScope`:

| Valor                      | Ámbito                                                                  |
| -------------------------- | ---------------------------------------------------------------------- |
| `main` (valor predeterminado de la configuración)    | Todos los mensajes directos comparten una sesión.                                             |
| `per-channel-peer`         | Cada par de canal y remitente obtiene un contexto de mensajes directos aislado (modo seguro de mensajes directos). |
| `per-account-channel-peer` | Como el anterior, pero con una separación adicional por cuenta (canales con varias cuentas).         |
| `per-peer`                 | Cada remitente obtiene una sesión para todos los canales del mismo tipo.     |

La incorporación mediante la CLI local conserva un valor `session.dmScope` explícito y, en caso contrario, lo deja sin definir, por lo que se aplica el valor predeterminado `"main"`: todos los mensajes directos de los distintos canales comparten la sesión principal continua del agente (el valor predeterminado para un agente personal). Para bandejas de entrada compartidas o multiusuario, establezca `session.dmScope: "per-channel-peer"`; `openclaw security audit` recomienda el aislamiento cuando detecta tráfico de mensajes directos de varios usuarios.

Este es un límite de contexto de mensajería, no un límite de administración del host. Si los usuarios son mutuamente hostiles y comparten el mismo host o la misma configuración del Gateway, ejecute gateways independientes para cada límite de confianza.

Si la misma persona se comunica mediante varios canales, utilice `session.identityLinks` para combinar esas sesiones de mensajes directos en una única identidad canónica. Consulte [Gestión de sesiones](/es/concepts/session) y [Configuración](/es/gateway/configuration).

## Visibilidad del contexto frente a autorización de activación

Son dos conceptos distintos:

- **Autorización de activación**: quién puede activar el agente (`dmPolicy`, `groupPolicy`, listas de permitidos y requisitos de mención).
- **Visibilidad del contexto**: qué contexto complementario recibe el modelo (cuerpo de la respuesta, texto citado, historial del hilo y metadatos reenviados).

`contextVisibility` controla el segundo:

- `"all"` (valor predeterminado): el contexto complementario se conserva tal como se recibe.
- `"allowlist"`: el contexto complementario se filtra para incluir únicamente remitentes admitidos por las comprobaciones activas de las listas de permitidos.
- `"allowlist_quote"`: como `allowlist`, pero conserva una respuesta citada explícita.

Configúrelo por canal o por sala o conversación; consulte [Grupos](/es/channels/groups#context-visibility-and-allowlists). Los informes que solo demuestran que «el modelo puede ver texto citado o histórico de remitentes no incluidos en la lista de permitidos» son hallazgos de refuerzo que pueden resolverse mediante `contextVisibility`, no omisiones de la autenticación ni del aislamiento por sí mismos; un informe con impacto en la seguridad debe demostrar además que se ha eludido un límite de confianza.

## Inyección de instrucciones

Un atacante crea un mensaje que manipula el modelo para que realice una acción insegura («ignora tus instrucciones», «vuelca tu sistema de archivos», «sigue este enlace y ejecuta comandos»). La inyección de instrucciones **no se resuelve** únicamente mediante mecanismos de protección en las instrucciones del sistema: estos solo proporcionan orientación flexible; la aplicación estricta proviene de la política de herramientas, las aprobaciones de ejecución, el aislamiento y las listas de canales permitidos (que los operadores pueden deshabilitar intencionalmente).

La inyección de instrucciones no requiere mensajes directos públicos: aunque solo el operador pueda enviar mensajes al bot, cualquier **contenido que no sea de confianza** que este lea (resultados de búsquedas u obtenciones web, páginas del navegador, correos electrónicos, documentos, archivos adjuntos y registros o código pegados) puede contener instrucciones maliciosas. El propio contenido constituye una superficie de amenaza, no solo el remitente.

Señales de alerta que deben tratarse como contenido no fiable:

- «Lee este archivo o URL y haz exactamente lo que indica».
- «Ignora las instrucciones del sistema o las reglas de seguridad».
- «Revela las instrucciones ocultas o los resultados de las herramientas».
- «Pega el contenido completo de ~/.openclaw o de los registros».

Medidas útiles en la práctica:

- Mantenga restringidos los mensajes directos entrantes (emparejamiento/listas de permitidos); prefiera el requisito de mención en los grupos; evite bots siempre activos en salas públicas.
- Trate de forma predeterminada los enlaces, los archivos adjuntos y las instrucciones pegadas como contenido hostil.
- Ejecute las herramientas sensibles en un entorno aislado; mantenga los secretos fuera del sistema de archivos accesible para el agente. El aislamiento es opcional: si el modo de aislamiento está desactivado, el valor implícito `host=auto` se resuelve al host del gateway, mientras que el valor explícito `host=sandbox` sigue aplicando un cierre seguro (no hay ningún entorno de aislamiento disponible). Establezca `host=gateway` para hacer explícito ese comportamiento en la configuración.
- Limite las herramientas de alto riesgo (`exec`, `browser`, `web_fetch`, `web_search`) a agentes de confianza o listas de permitidos explícitas.
- Si incluye intérpretes en la lista de permitidos (`python`, `node`, `ruby`, `perl`, `php`, `lua`, `osascript`), habilite `tools.exec.strictInlineEval` para que las formas de evaluación en línea (`-c`, `-e` y similares) sigan requiriendo aprobación explícita. En el modo de lista de permitidos, cualquier segmento heredoc (`<<`) requiere siempre la aprobación de un revisor o una aprobación explícita, independientemente de las comillas: un comando permitido no puede utilizar el cuerpo de un heredoc para omitir la revisión de la lista de permitidos.
- Reduzca el radio de impacto mediante un **agente lector** de solo lectura o sin herramientas que resuma el contenido que no sea de confianza y, después, transfiera el resumen al agente principal.
- Para los hooks de Gmail, la sesión integrada por mensaje aísla el contexto de la conversación, pero no elimina los permisos de herramientas o del espacio de trabajo del agente de destino. Dirija el correo que no sea de confianza a un agente lector específico, aplique [restricciones de aislamiento y herramientas por agente](/es/tools/multi-agent-sandbox-tools) y limite cualquier transferencia al agente principal mediante [`tools.agentToAgent`](/es/gateway/config-tools#toolsagenttoagent). Consulte [Integración con Gmail](/es/gateway/configuration-reference#gmail-integration).
- Mantenga desactivados `web_search` / `web_fetch` / `browser` para los agentes con herramientas habilitadas, salvo que sean necesarios.
- Para las entradas URL de OpenResponses (`input_file` / `input_image`), configure valores estrictos para `gateway.http.endpoints.responses.files.urlAllowlist` / `images.urlAllowlist` y mantenga un valor bajo para `maxUrlParts` (las listas de permitidos vacías se consideran no configuradas). Utilice `files.allowUrl: false` / `images.allowUrl: false` para deshabilitar por completo la obtención de URL.
- Mantenga los secretos fuera de las instrucciones; páselos mediante el entorno o la configuración en el host del gateway.

**La elección del modelo importa.** La resistencia a la inyección de prompts no es uniforme entre los distintos niveles de modelos: los modelos más pequeños o baratos son más susceptibles al uso indebido de herramientas y al secuestro de instrucciones mediante prompts adversarios.

<Warning>
Para los agentes con herramientas habilitadas o los que leen contenido no confiable, el riesgo de inyección de prompts con modelos más antiguos o pequeños suele ser demasiado alto. No ejecute esas cargas de trabajo con niveles de modelos débiles.
</Warning>

- Use el modelo de última generación y del mejor nivel para cualquier bot que pueda ejecutar herramientas o acceder a archivos o redes.
- No use niveles más antiguos, débiles o pequeños para agentes con herramientas habilitadas ni para bandejas de entrada no confiables.
- Si debe usar un modelo más pequeño, reduzca el radio de impacto: herramientas de solo lectura, aislamiento robusto, acceso mínimo al sistema de archivos y listas de permitidos estrictas. Habilite el aislamiento para todas las sesiones y deshabilite `web_search`/`web_fetch`/`browser`, salvo que las entradas estén estrictamente controladas.
- Para asistentes personales exclusivamente de chat, con entradas confiables y sin herramientas, los modelos más pequeños suelen ser adecuados.

### Encapsulado de contenido externo y entradas no confiables

El texto `input_file` de OpenResponses se sigue inyectando como contenido externo no confiable aunque el Gateway lo decodifique localmente: el bloque contiene marcadores de límite `<<<EXTERNAL_UNTRUSTED_CONTENT ...>>>` y metadatos `Source: External` (esta ruta omite el aviso `SECURITY NOTICE:` más largo que se usa en otros lugares). El mismo encapsulado basado en marcadores se aplica cuando la comprensión multimedia extrae texto de documentos adjuntos antes de incorporarlo al prompt multimedia.

OpenClaw también elimina del contenido externo encapsulado y de los metadatos los literales comunes de tokens especiales de las plantillas de chat de LLM autoalojados (tokens de rol o turno de Qwen/ChatML, Llama, Gemma, Mistral, Phi y GPT-OSS) antes de que lleguen al modelo. Los backends autoalojados compatibles con OpenAI (vLLM, SGLang, TGI, LM Studio y pilas personalizadas de tokenizadores de Hugging Face) a veces tokenizan cadenas literales como `<|im_start|>` o `<|start_header_id|>` como tokens estructurales de la plantilla de chat dentro del contenido del usuario; sin esta depuración, el texto no confiable de una página obtenida, del cuerpo de un correo electrónico o de la salida de una herramienta de contenido de archivos podría falsificar un límite de rol sintético `assistant`/`system`. La depuración se realiza en la capa de encapsulado de contenido externo, por lo que se aplica uniformemente a las herramientas de obtención y lectura y al contenido entrante de los canales. Los proveedores alojados (OpenAI y Anthropic) ya aplican su propia depuración en las solicitudes; mantenga habilitado el encapsulado de contenido externo y, cuando estén disponibles, prefiera configuraciones del backend que separen o escapen los tokens especiales.

Las respuestas salientes del modelo cuentan con un depurador independiente que elimina `<tool_call>`, `<function_calls>`, `<system-reminder>`, `<previous_response>` filtrados y otros componentes internos similares de las respuestas visibles para el usuario en el límite final de entrega al canal.

Esto no sustituye a `dmPolicy`, las listas de permitidos, las aprobaciones de ejecución, el aislamiento ni `contextVisibility`; corrige una omisión específica en la capa del tokenizador.

### Indicadores de omisión (manténgalos desactivados en producción)

- `hooks.mappings[].allowUnsafeExternalContent`
- `hooks.gmail.allowUnsafeExternalContent`
- Campo de carga útil de Cron `allowUnsafeExternalContent`

Habilítelos solo temporalmente para una depuración de alcance estricto; si se habilitan, aísle al agente (aislamiento + herramientas mínimas + espacio de nombres de sesión dedicado).

Las cargas útiles de los hooks son contenido no confiable incluso cuando la entrega procede de sistemas bajo su control (el contenido de correos, documentos o la Web puede contener inyecciones de prompts). Los niveles de modelos débiles aumentan este riesgo: para la automatización controlada por hooks, prefiera niveles de modelos modernos y robustos, mantenga una política de herramientas estricta (`tools.profile: "messaging"` o más restrictiva) y use aislamiento siempre que sea posible.

### Razonamiento y salida detallada en grupos

`/reasoning`, `/verbose` y `/trace` pueden exponer razonamientos internos, salidas de herramientas o diagnósticos de plugins que no están destinados a un canal público; pueden incluir argumentos de herramientas, URL, diagnósticos de plugins y datos que el modelo haya visto. Manténgalos deshabilitados en salas públicas; habilítelos únicamente en mensajes directos confiables o salas estrictamente controladas.

## Autorización de comandos

Los comandos con barra y las directivas solo se respetan para remitentes autorizados, determinados a partir de las listas de permitidos o el emparejamiento del canal y de `commands.useAccessGroups` (consulte [Configuración](/es/gateway/configuration) y [Comandos con barra](/es/tools/slash-commands)). Si la lista de permitidos de un canal está vacía o incluye `"*"`, los comandos quedan efectivamente abiertos para ese canal.

`/exec` es una función práctica exclusiva de la sesión para operadores autorizados; no escribe la configuración ni modifica otras sesiones.

## Herramientas del plano de control

Dos herramientas integradas siguen siendo sensibles para el plano de control:

- `gateway` lee la configuración con `config.schema.lookup` / `config.get`. No puede escribir la configuración, actualizar OpenClaw ni reiniciar el Gateway.
- `cron` crea trabajos programados que continúan ejecutándose después de que finaliza el chat o la tarea original.

La herramienta `gateway` permanece restringida al propietario porque la lectura de la configuración puede exponer secretos y la topología del host. Los agentes solicitan cambios persistentes de configuración o del ciclo de vida mediante la herramienta de delegación `openclaw`; OpenClaw los asigna a operaciones tipadas y exige aprobación humana antes de aplicarlos. Consulte [Agente de configuración de OpenClaw](/es/cli/openclaw#operations-and-approval).

Para cualquier agente o superficie que gestione contenido no confiable, deniegue estas herramientas de forma predeterminada:

```json5
{
  tools: {
    deny: ["gateway", "cron", "sessions_spawn", "sessions_send"],
  },
}
```

`commands.restart=false` deshabilita `/restart` y las solicitudes externas de reinicio `SIGUSR1`. La herramienta de agente `gateway` no dispone de una acción de reinicio.

## Ejecución de Node (`system.run`)

Si hay un nodo de macOS emparejado, el Gateway puede invocar `system.run` en él; esto constituye ejecución remota de código en ese Mac.

- Requiere emparejamiento del nodo (aprobación + token). El emparejamiento establece la identidad y la confianza del nodo, y emite el token; no es una superficie de aprobación por comando.
- El Gateway aplica una política global general de comandos del nodo mediante `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`. La lista de denegación solo coincide con nombres exactos de comandos del nodo (por ejemplo, `system.run`), no con texto de shell incluido en la carga útil de un comando; que un nodo que se vuelve a conectar anuncie una lista distinta de comandos no constituye por sí solo una vulnerabilidad si la política global del Gateway y las propias aprobaciones de ejecución del nodo siguen imponiendo el límite.
- La política `system.run` de cada nodo es el propio archivo de aprobaciones de ejecución del nodo (`exec.approvals.node.*`), controlado en el Mac mediante Settings -> Exec approvals (security + ask + allowlist); puede ser más estricta o más permisiva que la política global de identificadores de comandos del Gateway.
- Un nodo que ejecuta `security="full"` y `ask="off"` sigue el modelo predeterminado de operador confiable; es el comportamiento esperado, no un error, salvo que la implementación requiera una postura más estricta.
- El modo de aprobación vincula el contexto exacto de la solicitud y, cuando es posible, un único operando concreto de archivo o script local. Si OpenClaw no puede identificar exactamente un archivo local directo para un comando de intérprete o entorno de ejecución, se deniega la ejecución respaldada por aprobación en lugar de prometer una cobertura semántica completa.
- Para `host=node`, las ejecuciones respaldadas por aprobación también almacenan un `systemRunPlan` preparado y canónico; los reenvíos aprobados posteriores reutilizan ese plan almacenado, y la validación del Gateway rechaza las modificaciones del invocador en el comando, el directorio de trabajo o el contexto de sesión después de crear la solicitud de aprobación.
- Para deshabilitar por completo la ejecución remota: establezca la seguridad en `deny` y elimine el emparejamiento del nodo de ese Mac.

## Skills dinámicas (observador / nodos remotos)

OpenClaw puede actualizar la lista de Skills durante una sesión: el observador de Skills actualiza la instantánea en el siguiente turno del agente cuando cambia `SKILL.md`, y la conexión de un nodo de macOS puede hacer que las Skills exclusivas de macOS sean aptas (según el sondeo de binarios). Trate las carpetas de Skills como código confiable y restrinja quién puede modificarlas.

## Plugins

Los plugins se ejecutan dentro del proceso del Gateway; trátelos como código confiable.

- Instale únicamente desde fuentes de confianza; prefiera listas de permitidos `plugins.allow` explícitas; revise la configuración del plugin antes de habilitarlo; reinicie el Gateway después de modificar plugins.
- La instalación y actualización de plugins ejecuta código:
  - La ruta de instalación es el directorio de cada plugin bajo la raíz activa de instalación de plugins.
  - Los paquetes de ClawHub y el catálogo integrado u oficial de OpenClaw son fuentes confiables. Una nueva fuente arbitraria de npm, `npm-pack:`, git, ruta o archivo local, o marketplace muestra una advertencia antes de la instalación; las instalaciones no interactivas requieren `--force` después de revisar y confiar en esa fuente. `--force` confirma la procedencia y permite sobrescribir; no omite `security.installPolicy` ni las comprobaciones restantes de seguridad de instalación. Las actualizaciones reutilizan la fuente ya seleccionada.
  - OpenClaw no ejecuta bloqueos locales integrados de código peligroso durante la instalación o actualización. Use `security.installPolicy` para que el operador tome decisiones locales de permiso o bloqueo y `openclaw security audit --deep` para el análisis de diagnóstico.
  - Las instalaciones de plugins mediante npm y git ejecutan la convergencia de dependencias del gestor de paquetes únicamente durante el flujo explícito de instalación o actualización. Las rutas y los archivos locales se tratan como paquetes autosuficientes; OpenClaw los copia o referencia sin ejecutar `npm install`.
  - Prefiera versiones exactas fijadas (`@scope/pkg@1.2.3`) e inspeccione el código desempaquetado antes de habilitarlo.
  - `--dangerously-force-unsafe-install` está obsoleto y ya no modifica el comportamiento de instalación o actualización.
  - `security.installPolicy` permite a los operadores ejecutar un comando local confiable para tomar decisiones de permiso o bloqueo específicas del host en instalaciones de Skills y plugins. Se ejecuta después de preparar el material de origen, pero antes de continuar con la instalación, también se aplica a las Skills de ClawHub y los indicadores inseguros obsoletos no lo omiten.

Detalles: [Plugins](/es/tools/plugin)

## Aislamiento

Documento específico: [Aislamiento](/es/gateway/sandboxing)

Dos enfoques complementarios:

- **Gateway completo en Docker** (límite del contenedor): [Docker](/es/install/docker)
- **Aislamiento de herramientas** (`agents.defaults.sandbox`; Gateway del host + herramientas aisladas; Docker es el backend predeterminado): [Aislamiento](/es/gateway/sandboxing)

<Note>
Para impedir el acceso entre agentes, mantenga `agents.defaults.sandbox.scope` en `"agent"` (valor predeterminado) o use `"session"` para un aislamiento más estricto por sesión. `scope: "shared"` utiliza un único contenedor o espacio de trabajo.
</Note>

Acceso al espacio de trabajo del agente dentro del entorno aislado (`agents.defaults.sandbox.workspaceAccess`):

- `"none"` (valor predeterminado): las herramientas ven un espacio de trabajo aislado bajo `~/.openclaw/sandboxes`; el espacio de trabajo del agente no es accesible.
- `"ro"`: monta el espacio de trabajo del agente en modo de solo lectura en `/agent` (deshabilita `write`/`edit`/`apply_patch`).
- `"rw"`: monta el espacio de trabajo del agente en modo de lectura y escritura en `/workspace`.

Los `sandbox.docker.binds` adicionales se validan con respecto a rutas de origen normalizadas y canonicalizadas. Una lista de denegación de rutas bloqueadas abarca `/etc`, `/private/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot` y los directorios que suelen contener o servir como alias del socket de Docker (`/run`, `/var/run` y `docker.sock` dentro de ellos), además de las subrutas de credenciales de HOME (`.aws`, `.cargo`, `.config`, `.docker`, `.gnupg`, `.netrc`, `.npm`, `.ssh`). Los trucos con enlaces simbólicos en directorios superiores y los alias canónicos del directorio de inicio se resuelven a través de los antecesores existentes y se vuelven a comprobar, por lo que continúan denegándose de forma segura si se resuelven dentro de una raíz bloqueada.

<Warning>
`tools.elevated` es el mecanismo de escape global de referencia que ejecuta comandos fuera del entorno aislado. El host efectivo es `gateway` de forma predeterminada, o `node` cuando el destino de ejecución está configurado como `node`. Mantenga `tools.elevated.allowFrom` estrictamente restringido y no lo habilite para desconocidos. Restrínjalo aún más por agente mediante `agents.entries.*.tools.elevated`. Consulte [Modo elevado](/es/tools/elevated).
</Warning>

### Mecanismo de protección para la delegación a subagentes

Si se permiten herramientas de sesión, las ejecuciones delegadas de subagentes deben tratarse como otra decisión de límites:

- Deniegue `sessions_spawn` a menos que el agente realmente necesite delegar.
- Mantenga `agents.defaults.subagents.allowAgents` y cualquier anulación `agents.entries.*.subagents.allowAgents` por agente restringidos a agentes de destino cuya seguridad sea conocida.
- Para los flujos de trabajo que deban permanecer en un entorno aislado, llame a `sessions_spawn` con `sandbox: "require"` (el valor predeterminado es `"inherit"`); `"require"` falla de inmediato cuando el entorno de ejecución secundario de destino no está aislado.

### Modo de solo lectura

Cree un perfil de solo lectura combinando `agents.defaults.sandbox.workspaceAccess: "ro"` (o `"none"` para impedir el acceso al espacio de trabajo) con listas de herramientas permitidas y denegadas que bloqueen `write`, `edit`, `apply_patch`, `exec`, `process`, etc.

- `tools.exec.applyPatch.workspaceOnly: true` (valor predeterminado): impide que `apply_patch` escriba o elimine fuera del directorio del espacio de trabajo incluso con el aislamiento desactivado. Establezca `false` solo si desea intencionadamente que `apply_patch` modifique archivos fuera del espacio de trabajo.
- `tools.fs.workspaceOnly: true` (opcional): restringe las rutas de `read`/`write`/`edit`/`apply_patch` y las rutas de carga automática de imágenes de solicitudes nativas al directorio del espacio de trabajo.
- Mantenga restringidas las raíces del sistema de archivos: evite raíces amplias, como el directorio personal, para los espacios de trabajo del agente o del entorno aislado, ya que pueden exponer archivos locales confidenciales (por ejemplo, el estado o la configuración en `~/.openclaw`) a las herramientas del sistema de archivos.

## Perfiles de acceso por agente (varios agentes)

Cada agente puede tener su propia política de aislamiento y herramientas: acceso completo, solo lectura o ningún acceso. Consulte [Entorno aislado y herramientas para varios agentes](/es/tools/multi-agent-sandbox-tools) para conocer las reglas de precedencia.

Patrones habituales: agente personal (acceso completo, sin aislamiento), agente familiar o de trabajo (aislado y con herramientas de solo lectura), agente público (aislado y sin herramientas del sistema de archivos ni del shell).

### Acceso completo (sin aislamiento)

```json5
{
  agents: {
    list: [
      { id: "personal", workspace: "~/.openclaw/workspace-personal", sandbox: { mode: "off" } },
    ],
  },
}
```

### Herramientas de solo lectura y espacio de trabajo de solo lectura

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: ["read"],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

### Sin acceso al sistema de archivos ni al shell (se permite la mensajería del proveedor)

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          // Las herramientas de sesión pueden revelar datos de las transcripciones. El ámbito predeterminado es el actual y los generados;
          // las lecturas también incluyen los grupos del mismo agente observados mediante el conocimiento ambiental de grupos.
          // Use visibility: "self" para excluir esas sesiones observadas.
          sessions: { visibility: "tree" }, // self | tree | agent | all
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "discord",
            "slack",
            "telegram",
            "whatsapp",
          ],
          deny: [
            "apply_patch",
            "browser",
            "canvas",
            "cron",
            "edit",
            "exec",
            "gateway",
            "image",
            "nodes",
            "process",
            "read",
            "write",
          ],
        },
      },
    ],
  },
}
```

## Riesgos del control del navegador

Habilitar el control del navegador proporciona al modelo un navegador real. Si ese perfil ya tiene sesiones iniciadas, el modelo puede acceder a esas cuentas y datos; los perfiles del navegador deben tratarse como estado confidencial.

- Prefiera un perfil dedicado para el agente (el perfil `openclaw` predeterminado); evite el perfil personal que utiliza a diario.
- Mantenga desactivado el control del navegador del host para los agentes aislados, salvo que sean de confianza.
- La API independiente de control del navegador por bucle invertido solo admite la autenticación mediante secreto compartido (autenticación de portador con token del Gateway o contraseña del Gateway); no utiliza los encabezados de identidad de un proxy de confianza ni de Tailscale Serve.
- Trate las descargas del navegador como entradas no confiables; prefiera un directorio de descargas aislado.
- Desactive la sincronización del navegador y los gestores de contraseñas en el perfil del agente si es posible.
- Para gateways remotos, el «control del navegador» equivale al «acceso de operador» a todo aquello a lo que pueda acceder ese perfil.
- Mantenga los hosts del Gateway y de los nodos accesibles únicamente desde la tailnet; evite exponer los puertos de control del navegador a la LAN o a la Internet pública.
- Desactive el enrutamiento del proxy del navegador cuando no sea necesario (`gateway.nodes.browser.mode="off"`).
- El modo de sesión existente de Chrome MCP no es «más seguro»: puede actuar en su nombre en todo aquello a lo que pueda acceder el perfil de Chrome de ese host.
- Ejecute un **host de nodo** en el equipo del navegador y permita que el Gateway retransmita las acciones del navegador cuando el Gateway sea remoto respecto al navegador (consulte [Herramienta de navegador](/es/tools/browser)); trate el emparejamiento de nodos como acceso de administrador, mantenga el Gateway y el host de nodo en la misma tailnet y evite exponer los puertos de retransmisión o control a través de la LAN, la Internet pública o Tailscale Funnel.

### Política SSRF del navegador (estricta de forma predeterminada)

Los destinos privados o internos permanecen bloqueados a menos que se habiliten explícitamente.

- Valor predeterminado: `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` sin establecer, por lo que los destinos privados, internos o de uso especial permanecen bloqueados. Se sigue aceptando el alias heredado `allowPrivateNetwork`.
- Habilitación explícita: establezca `dangerouslyAllowPrivateNetwork: true` para permitir esos destinos.
- En el modo estricto, use `hostnameAllowlist` (patrones como `*.example.com`) y `allowedHostnames` (excepciones de host exactas, incluidos nombres que de otro modo estarían bloqueados, como `localhost`) para establecer excepciones explícitas.
- Las solicitudes de navegación directa se comprueban previamente. Durante la acción y el período de gracia limitado posterior, las interacciones protegidas de Playwright (clic, clic por coordenadas, desplazamiento del puntero, arrastre, desplazamiento, selección, pulsación, escritura, cumplimentación de formularios y evaluación) interceptan las cargas de documentos de nivel superior y de submarcos denegadas por la política antes de transmitir los bytes de la solicitud HTTP y, a continuación, vuelven a comprobar con el mejor esfuerzo la URL `http(s)` final.
- Antes de cada inicio nuevo de Chrome administrado, OpenClaw intenta desactivar la predicción de red, lo que impide las preconexiones especulativas observadas de Chromium para esas cargas denegadas. Esta es una defensa en profundidad, no un límite de la política: un navegador reutilizado tras reiniciar el servicio de control y otros motores de navegador podrían no compartir esta protección. El enrutamiento de páginas sigue siendo una interceptación en el nivel de las solicitudes, no un cortafuegos de red: los saltos de redirección, la primera solicitud de una ventana emergente, el tráfico de Service Worker, el código de página que se ejecuta después del período limitado de protección y algunas rutas de segundo plano o de subrecursos pueden eludirla. Las comprobaciones de la URL final siguen siendo una defensa de detección y cuarentena; la prevención completa requiere aislamiento de salida por parte del propietario o un proxy que aplique la política.

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"],
    },
  },
}
```

## Exposición de red

### Enlace, puerto y cortafuegos

El Gateway multiplexa WebSocket y HTTP en un solo puerto (valor predeterminado: `18789`; configuración, indicadores y variables de entorno: `gateway.port`, `--port`, `OPENCLAW_GATEWAY_PORT`). Esa superficie HTTP incluye la interfaz de control (recursos de la SPA, ruta base predeterminada `/`) y el host del lienzo (`/__openclaw__/canvas` y `/__openclaw__/a2ui`: HTML/JS arbitrario; trátelo como contenido no confiable cuando se cargue en un navegador normal; no lo exponga a redes o usuarios no confiables ni comparta un origen con superficies web privilegiadas).

`gateway.bind` controla dónde escucha el Gateway:

- `"loopback"` (valor predeterminado): solo pueden conectarse los clientes locales.
- `"lan"`, `"tailnet"`, `"custom"`: amplían la superficie de ataque. Úselos únicamente con autenticación del Gateway (token o contraseña compartidos, o un proxy de confianza configurado correctamente) y un cortafuegos real.

Reglas generales: prefiera Tailscale Serve a los enlaces de LAN (Serve mantiene el Gateway en el bucle invertido y Tailscale gestiona el acceso); si debe enlazarlo a la LAN, limite el puerto mediante el cortafuegos a una lista estricta de direcciones IP de origen permitidas en lugar de reenviarlo ampliamente; nunca exponga el Gateway sin autenticación en `0.0.0.0`.

### Publicación de puertos de Docker con UFW

Los puertos de contenedores publicados (`-p HOST:CONTAINER` o `ports:` de Compose) se enrutan mediante las cadenas de reenvío de Docker, no solo mediante las reglas `INPUT` del host. Aplique las reglas en `DOCKER-USER` (se evalúan antes que las propias reglas de aceptación de Docker); la mayoría de las distribuciones modernas utilizan el frontend `iptables-nft`, que también aplica estas reglas al backend nftables.

```bash
# /etc/ufw/after.rules (añadir como sección *filter independiente)
*filter
:DOCKER-USER - [0:0]
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
-A DOCKER-USER -s 127.0.0.0/8 -j RETURN
-A DOCKER-USER -s 10.0.0.0/8 -j RETURN
-A DOCKER-USER -s 172.16.0.0/12 -j RETURN
-A DOCKER-USER -s 192.168.0.0/16 -j RETURN
-A DOCKER-USER -s 100.64.0.0/10 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j RETURN
-A DOCKER-USER -p tcp --dport 443 -j RETURN
-A DOCKER-USER -m conntrack --ctstate NEW -j DROP
-A DOCKER-USER -j RETURN
COMMIT
```

IPv6 tiene tablas independientes; añada una política equivalente en `/etc/ufw/after6.rules` si IPv6 de Docker está habilitado. Evite codificar de forma fija los nombres de las interfaces (`eth0`), ya que varían entre las imágenes de VPS (`ens3`, `enp*`, etc.) y una discrepancia puede omitir silenciosamente la regla de denegación.

```bash
ufw reload
iptables -S DOCKER-USER
ip6tables -S DOCKER-USER
nmap -sT -p 1-65535 <public-ip> --open
```

Los puertos externos esperados deben ser únicamente los que se expongan intencionadamente (para la mayoría de las configuraciones: SSH y los puertos del proxy inverso).

### Detección mDNS/Bonjour

Cuando el plugin `bonjour` incluido está habilitado, el Gateway anuncia su presencia mediante mDNS (`_openclaw-gw._tcp`, puerto 5353) para detectar dispositivos locales. El modo completo incluye registros TXT que exponen detalles operativos: `cliPath` (ruta del sistema de archivos que revela el nombre de usuario y la ubicación de instalación), `sshPort` (anuncia la disponibilidad de SSH), `displayName`/`lanHost` (información del nombre de host). La difusión de detalles de infraestructura facilita el reconocimiento de la LAN.

- Mantenga Bonjour deshabilitado salvo que se necesite la detección en la LAN: se inicia automáticamente en hosts macOS y requiere habilitación explícita en otros sistemas; las URL directas del Gateway, la tailnet, SSH o DNS-SD de área extensa evitan la multidifusión local.
- El **modo mínimo** (predeterminado cuando Bonjour está habilitado y recomendado para gateways expuestos) omite los campos confidenciales:

  ```json5
  { discovery: { mdns: { mode: "minimal" } } }
  ```

- El modo **desactivado** impide la detección local mientras mantiene habilitado el plugin:

  ```json5
  { discovery: { mdns: { mode: "off" } } }
  ```

- El **modo completo** (habilitación explícita) incluye `cliPath` y `sshPort`:

  ```json5
  { discovery: { mdns: { mode: "full" } } }
  ```

- También se puede establecer `OPENCLAW_DISABLE_BONJOUR=1` para deshabilitar mDNS sin modificar la configuración.

En el modo mínimo, el Gateway anuncia `role`, `gatewayPort`, `transport`, pero omite `cliPath`/`sshPort`; las aplicaciones que necesiten la ruta de la CLI pueden obtenerla a través de la conexión WebSocket autenticada.

### Autenticación WebSocket del Gateway

La autenticación del Gateway es obligatoria de forma predeterminada: si no se configura ninguna vía de autenticación válida, el Gateway rechaza las conexiones WebSocket (cierre seguro). La incorporación genera un token de forma predeterminada (incluso para el bucle invertido), por lo que los clientes locales deben autenticarse.

```json5
{ gateway: { auth: { mode: "token", token: "your-token" } } }
```

`openclaw doctor --generate-gateway-token` puede generar uno.

<Note>
`gateway.remote.token` y `gateway.remote.password` son fuentes de credenciales de cliente; por sí solas no protegen el acceso WS local. Las rutas de llamadas locales usan `gateway.remote.*` solo como alternativa cuando `gateway.auth.*` no está definido. Si `gateway.auth.token` o `gateway.auth.password` se configura explícitamente mediante SecretRef y no se puede resolver, la resolución falla de forma cerrada (sin que la alternativa remota enmascare el error).
</Note>

Fije el TLS remoto con `gateway.remote.tlsFingerprint` al usar `wss://`. Se acepta `ws://` sin cifrar para loopback, literales de IP privadas, `.local` y URL de gateway `*.ts.net` de Tailnet; para otros nombres DNS privados de confianza, defina `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` en el proceso cliente como medida de emergencia (solo en el entorno del proceso, no como clave de `openclaw.json`). El emparejamiento móvil y las rutas manuales/escaneadas del gateway en Android son más estrictos: solo permiten texto sin cifrar para loopback, mientras que la LAN privada, las direcciones link-local, `.local` y los nombres de host sin punto deben usar TLS, salvo que se habilite explícitamente la ruta de texto sin cifrar para redes privadas de confianza.

El emparejamiento de dispositivos se aprueba automáticamente para las conexiones locales directas por loopback (además de una ruta limitada de autoconexión local del backend/contenedor para flujos auxiliares de secreto compartido de confianza); las conexiones de Tailnet y LAN, incluidas las conexiones desde el mismo host a una dirección de Tailnet, se consideran remotas y siguen requiriendo aprobación. Una dirección `tailnet` resuelta o una dirección `custom` distinta de `127.0.0.1` o `0.0.0.0` añade un listener `127.0.0.1` independiente; solo las conexiones a ese listener local reciben semántica de loopback. La presencia de indicios de encabezados reenviados en una solicitud de loopback invalida la localidad de loopback; la aprobación automática de actualizaciones de metadatos tiene un alcance limitado. Consulte [Emparejamiento del Gateway](/es/gateway/pairing).

Modos de autenticación:

- `"token"`: token bearer compartido (recomendado para la mayoría de las configuraciones).
- `"password"`: es preferible definirla mediante `OPENCLAW_GATEWAY_PASSWORD`.
- `"trusted-proxy"`: confía en un proxy inverso con reconocimiento de identidad para autenticar a los usuarios y transmitir la identidad mediante encabezados. Consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).

Lista de comprobación para la rotación (token/contraseña): genere o defina un secreto nuevo (`gateway.auth.token` o `OPENCLAW_GATEWAY_PASSWORD`); reinicie el Gateway (o la aplicación de macOS si supervisa el Gateway); actualice los clientes remotos (`gateway.remote.token`/`.password`); verifique que las credenciales anteriores ya no funcionen.

### Encabezados de identidad de Tailscale Serve

Cuando `gateway.auth.allowTailscale` es `true` (valor predeterminado para Serve), OpenClaw acepta el encabezado de identidad de Tailscale Serve `tailscale-user-login` para la autenticación de Control UI/WebSocket. Verifica la identidad resolviendo la dirección `x-forwarded-for` mediante el daemon local de Tailscale (`tailscale whois`) y comparándola con el encabezado; esto solo se activa para solicitudes de loopback que contienen `x-forwarded-for`, `x-forwarded-proto` y `x-forwarded-host` inyectados por Tailscale. Para esta comprobación asíncrona, los intentos fallidos del mismo `{scope, ip}` se serializan antes de que el limitador registre el fallo, por lo que los reintentos incorrectos simultáneos de un cliente Serve pueden bloquear de inmediato el segundo intento.

Los endpoints de la API HTTP (`/v1/*`, `/tools/invoke`, `/api/channels/*`) no usan la autenticación mediante encabezados de identidad de Tailscale; siguen el modo de autenticación HTTP configurado en el gateway.

La autenticación bearer HTTP del Gateway proporciona, en la práctica, acceso de operador de todo o nada. Las credenciales que pueden llamar a `/v1/chat/completions`, `/v1/responses`, rutas de plugins como `/api/v1/admin/rpc` o `/api/channels/*` son secretos de operador con acceso completo para ese gateway: la autenticación bearer mediante secreto compartido restaura todos los ámbitos predeterminados del operador (`operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`) y la semántica de propietario para los turnos del agente, y los valores más restringidos de `x-openclaw-scopes` no reducen esa ruta de secreto compartido. La semántica de ámbitos por solicitud solo se aplica cuando la solicitud procede de un modo con identidad (autenticación mediante proxy de confianza) o de una entrada privada explícitamente sin autenticación; en esos modos, omitir `x-openclaw-scopes` recurre al conjunto normal de ámbitos predeterminados del operador, y los encabezados de nivel de propietario como `x-openclaw-model` requieren `operator.admin` cuando se restringen los ámbitos. `/tools/invoke` y los endpoints HTTP del historial de sesiones siguen la misma regla de secreto compartido. No comparta estas credenciales con solicitantes que no sean de confianza; es preferible usar gateways distintos para cada límite de confianza.

La autenticación de Serve sin token presupone que el propio host del gateway es de confianza; no protege contra procesos hostiles en el mismo host. Si puede ejecutarse código local que no sea de confianza en el host del gateway, deshabilite `allowTailscale` y exija autenticación explícita mediante secreto compartido (`token` o `password`).

No reenvíe estos encabezados desde su propio proxy inverso. Si termina TLS o utiliza un proxy delante del gateway, deshabilite `allowTailscale` y use en su lugar autenticación mediante secreto compartido o [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).

Consulte [Tailscale](/es/gateway/tailscale) y la [descripción general de la web](/es/web).

### Configuración del proxy inverso

Defina `gateway.trustedProxies` para gestionar correctamente la IP reenviada del cliente detrás de nginx/Caddy/Traefik/etc. Cuando el Gateway detecta encabezados de proxy procedentes de una dirección que **no** está en `trustedProxies`, no considera local la conexión; si la autenticación del gateway está deshabilitada, la conexión se rechaza. Esto evita que las conexiones a través de proxy parezcan proceder de localhost y reciban confianza automática.

`trustedProxies` también proporciona datos a `gateway.auth.mode: "trusted-proxy"`, que es más estricto: de forma predeterminada, falla de forma cerrada con proxies cuyo origen es loopback. Los proxies inversos de loopback en el mismo host pueden usar `trustedProxies` para detectar clientes locales y gestionar las IP reenviadas, pero solo pueden satisfacer el modo de autenticación `trusted-proxy` cuando `gateway.auth.trustedProxy.allowLoopback = true`; en caso contrario, use autenticación mediante token/contraseña.

```yaml
gateway:
  trustedProxies:
    - "10.0.0.1" # IP del proxy inverso
  allowRealIpFallback: false # valor predeterminado: false; habilítelo solo si el proxy no puede proporcionar X-Forwarded-For
  auth:
    mode: password
    password: ${OPENCLAW_GATEWAY_PASSWORD}
```

Cuando se define `trustedProxies`, el Gateway usa `X-Forwarded-For` para determinar la IP del cliente; `X-Real-IP` se ignora salvo que se defina explícitamente `gateway.allowRealIpFallback: true`. Asegúrese de que el proxy **sobrescriba** `X-Forwarded-For`/`X-Real-IP` en lugar de añadir valores:

```nginx
# correcto
proxy_set_header X-Forwarded-For $remote_addr;
proxy_set_header X-Real-IP $remote_addr;

# incorrecto: conserva/añade valores no fiables proporcionados por el cliente
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Los encabezados de proxy de confianza no hacen que el emparejamiento de dispositivos Node se considere automáticamente de confianza: `gateway.nodes.pairing.autoApproveCidrs` es una política de operador independiente y deshabilitada de forma predeterminada, y las rutas de encabezados de proxy de confianza cuyo origen es loopback siguen excluidas de la aprobación automática de Node incluso cuando está habilitada la autenticación mediante proxy de confianza de loopback (porque los solicitantes locales pueden falsificar esos encabezados).

### Notas sobre HSTS y el origen

- El gateway de OpenClaw está diseñado principalmente para uso local/por loopback. Si termina TLS en un proxy inverso, configure HSTS allí.
- Si el propio gateway termina HTTPS, `gateway.http.securityHeaders.strictTransportSecurity` emite el encabezado HSTS en las respuestas de OpenClaw.
- Las implementaciones de Control UI fuera de loopback requieren `gateway.controlUi.allowedOrigins` de forma predeterminada; `allowedOrigins: ["*"]` es una política explícita que permite todos los orígenes, no una configuración predeterminada reforzada: evítela fuera de pruebas locales estrictamente controladas.
- Los fallos de autenticación de origen del navegador en loopback siguen sujetos a límites de frecuencia aunque esté habilitada la exención general para loopback, pero la clave de bloqueo tiene un ámbito por cada valor normalizado de `Origin`, en lugar de usar un único grupo compartido para localhost.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` habilita el modo alternativo de origen basado en el encabezado Host; debe considerarse una política peligrosa seleccionada por el operador.
- Considere la revinculación de DNS y el comportamiento de los encabezados de host del proxy como aspectos de refuerzo de la implementación; mantenga `trustedProxies` estrictamente limitado y evite exponer el gateway directamente a Internet.
- Guía detallada de implementación: [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth#tls-termination-and-hsts).

### Control UI mediante HTTP

Control UI necesita un contexto seguro (HTTPS o localhost) para generar la identidad del dispositivo.

- `gateway.controlUi.allowInsecureAuth`: opción de compatibilidad local. En localhost, permite la autenticación de Control UI sin identidad del dispositivo cuando la página se carga mediante HTTP no seguro. No omite las comprobaciones de emparejamiento ni relaja los requisitos de identidad del dispositivo remoto (fuera de localhost). Es preferible usar HTTPS (Tailscale Serve) o abrir la interfaz en `127.0.0.1`.
- `gateway.controlUi.dangerouslyDisableDeviceAuth`: entrada de emergencia retirada. Las configuraciones anteriores conservan el acceso autenticado de Control UI, limitado al emparejamiento, para realizar correcciones hasta que un navegador reabierto mediante HTTPS o localhost complete la migración de autoemparejamiento limitada y explícita; no la añada a la configuración actual.
- Independientemente de esas opciones, un `gateway.auth.mode: "trusted-proxy"` correcto puede admitir sesiones de Control UI de **operador** sin identidad del dispositivo; se trata de un comportamiento deliberado del modo de autenticación, no de un atajo de `allowInsecureAuth`, y no se extiende a las sesiones de Control UI con función de Node.

`openclaw security audit` muestra una advertencia cuando `allowInsecureAuth` está habilitado.

### Opciones no seguras/peligrosas

`openclaw security audit` genera `config.insecure_or_dangerous_flags` por cada opción conocida de depuración no segura/peligrosa que esté habilitada (un hallazgo por opción). Manténgalas sin definir en producción. Si se configuran supresiones de auditoría, `security.audit.suppressions.active` permanece en la salida activa incluso cuando los hallazgos coincidentes pasan a `suppressedFindings`.

<AccordionGroup>
  <Accordion title="Opciones que actualmente supervisa la auditoría">
    - `gateway.controlUi.allowInsecureAuth=true`
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
    - migración pendiente de la autenticación de dispositivos de Control UI importada de la opción retirada `gateway.controlUi.dangerouslyDisableDeviceAuth=true`
    - `security.audit.suppressions configured (<count>)`
    - `hooks.gmail.allowUnsafeExternalContent=true`
    - `hooks.mappings[<index>].allowUnsafeExternalContent=true`
    - `tools.exec.applyPatch.workspaceOnly=false`
    - `plugins.entries.acpx.config.permissionMode=approve-all`

  </Accordion>

  <Accordion title="Todas las claves dangerous*/dangerously* del esquema de configuración">
    Control UI y navegador:
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback`
    - `gateway.controlUi.dangerouslyDisableDeviceAuth` (entrada de actualización retirada)
    - `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`

    Coincidencia de nombres de canales (canales incluidos y de plugins; también por `accounts.<accountId>` cuando corresponda):
    - `channels.discord.dangerouslyAllowNameMatching`
    - `channels.googlechat.dangerouslyAllowNameMatching`
    - `channels.msteams.dangerouslyAllowNameMatching`
    - `channels.slack.dangerouslyAllowNameMatching`
    - `channels.irc.dangerouslyAllowNameMatching` (canal de plugin)
    - `channels.mattermost.dangerouslyAllowNameMatching` (canal de plugin)
    - `channels.synology-chat.dangerouslyAllowNameMatching` (canal de plugin)
    - `channels.synology-chat.dangerouslyAllowInheritedWebhookPath` (canal de plugin)
    - `channels.zalouser.dangerouslyAllowNameMatching` (canal de plugin)

    Exposición de red:
    - `channels.telegram.network.dangerouslyAllowPrivateNetwork` (también por cuenta)

    Docker del sandbox (valores predeterminados y por agente):
    - `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets`
    - `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources`
    - `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin`

  </Accordion>
</AccordionGroup>

## Implementación y confianza en el host

- Cifrado de disco completo en el host del Gateway; se recomienda una cuenta de usuario del sistema operativo dedicada para el Gateway si el host es compartido.
- Bloqueo de dependencias del paquete publicado: los checkouts del código fuente usan `pnpm-lock.yaml`; el paquete npm `openclaw` publicado y los paquetes de plugins npm propiedad de OpenClaw incluyen `npm-shrinkwrap.json`, de modo que las instalaciones utilicen el grafo de dependencias transitivas revisado de la versión en lugar de resolver un grafo nuevo durante la instalación. Este es un límite de refuerzo de la cadena de suministro y reproducibilidad de las versiones, no un entorno aislado; consulte [shrinkwrap de npm](/es/gateway/security/shrinkwrap).
- Operaciones de archivos seguras: OpenClaw usa `@openclaw/fs-safe` para el acceso a archivos restringido a la raíz, las escrituras atómicas, la extracción de archivos, los espacios de trabajo temporales y los auxiliares para archivos de secretos. El auxiliar opcional de Python para POSIX está **desactivado** de forma predeterminada; establezca `OPENCLAW_FS_SAFE_PYTHON_MODE=auto` o `require` solo si desea el refuerzo adicional de las mutaciones relativas a descriptores de archivo y puede disponer de un entorno de ejecución de Python. Detalles: [Operaciones de archivos seguras](/es/gateway/security/secure-file-operations).
- Riesgo de un espacio de trabajo compartido de Slack: si todos los usuarios de Slack pueden enviar mensajes al bot, el riesgo principal es la autoridad delegada sobre las herramientas: cualquier remitente permitido puede provocar llamadas a herramientas (`exec`, navegador, herramientas de red o archivos) dentro de la política del agente; la inyección de instrucciones o contenido por parte de un remitente puede afectar al estado, los dispositivos y las salidas compartidos; y, si el agente compartido tiene credenciales o archivos confidenciales, cualquier remitente permitido puede llegar a provocar la exfiltración mediante el uso de herramientas. Utilice agentes y gateways separados con un conjunto mínimo de herramientas para los flujos de trabajo de equipo; mantenga privados los agentes con datos personales.
- Agente compartido por la empresa (patrón aceptable): es adecuado cuando todos los usuarios del agente pertenecen al mismo límite de confianza (por ejemplo, un equipo de una empresa) y el agente está estrictamente limitado al ámbito empresarial. Ejecútelo en una máquina, máquina virtual o contenedor dedicados; utilice un usuario del sistema operativo y un navegador, perfil y cuentas dedicados; y no inicie sesión en ese entorno de ejecución con cuentas personales de Apple o Google ni con perfiles personales del gestor de contraseñas o del navegador. Mezclar identidades personales y empresariales en el mismo entorno de ejecución elimina la separación y aumenta el riesgo de exposición de datos personales.

## Secretos en disco

Suponga que cualquier elemento bajo `~/.openclaw/` (o `$OPENCLAW_STATE_DIR/`) puede contener secretos o datos privados:

| Ruta                                           | Contenido                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.json`                                | La configuración puede incluir tokens (del Gateway y del Gateway remoto), ajustes del proveedor y listas de permitidos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `credentials/**`                               | Credenciales de canales (por ejemplo, credenciales de WhatsApp), listas de permitidos para el emparejamiento e importaciones de OAuth heredadas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `state/openclaw.sqlite`                        | Estado compartido del entorno de ejecución, incluidos los tokens de acceso y actualización de OAuth de MCP nativo, los secretos de registro dinámico de clientes y el estado de descubrimiento.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | Estado del entorno de ejecución por agente, incluidos los perfiles de autenticación de modelos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `agents/<agentId>/agent/auth-profiles.json`    | Fuente heredada de migración de autenticación de modelos; doctor importa los registros compatibles en la base de datos SQLite por agente.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `agents/<agentId>/agent/codex-home/**`         | Cuenta del servidor de aplicaciones de Codex, configuración, Skills, Plugins, estado nativo de los hilos y diagnósticos por agente (valor predeterminado).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `$CODEX_HOME/**` o `~/.codex/**`              | Estado nativo del entorno de ejecución de Codex. El arnés ordinario solo accede a él con `plugins.entries.codex.config.appServer.homeScope: "user"` explícito. La conexión de supervisión independiente accede a él cuando su ámbito de inicio resuelto es `"user"`, que es el valor predeterminado para stdio o Unix cuando no está establecido. Contiene la cuenta nativa de Codex, la configuración, los Plugins y el almacén de hilos. La supervisión enumera los metadatos de origen y conserva la rama nativa canónica de un Chat continuado y los turnos posteriores en esa conexión; la ramificación copia un historial persistente acotado del usuario y del asistente en un Chat de OpenClaw autenticado y bloqueado a un modelo. Habilítelo únicamente para un Gateway controlado por el propietario. Consulte [arnés de Codex](/es/plugins/codex-harness#share-threads-with-codex-desktop-and-cli) y [supervisión de Codex](/es/plugins/codex-supervision). |
| `secrets.json` (opcional)                      | Carga útil de secretos respaldada por archivos que utilizan los proveedores SecretRef `file` (`secrets.providers`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `agents/<agentId>/agent/auth.json`             | Archivo de compatibilidad heredado; las entradas estáticas `api_key` se depuran cuando se detectan.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | Estado del entorno de ejecución por agente, incluidas las filas de sesiones y las transcripciones que pueden contener mensajes privados y la salida de herramientas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `agents/<agentId>/sessions/**`                 | Fuentes y archivos heredados de migración de sesiones que pueden contener mensajes privados y la salida de herramientas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| paquetes de Plugins incluidos                        | Plugins instalados (además de sus `node_modules/`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `sandboxes/**`                                 | Espacios de trabajo del entorno aislado de herramientas; pueden acumular copias de archivos leídos o escritos dentro del entorno aislado.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

### Mapa de almacenamiento de credenciales

También resulta útil para decidir qué incluir en las copias de seguridad:

- WhatsApp: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- Token del bot de Telegram: configuración/entorno o `channels.telegram.tokenFile` (solo archivos normales; se rechazan los enlaces simbólicos)
- Token del bot de Discord: configuración/entorno o SecretRef (proveedores de entorno/archivo/ejecución)
- Tokens de Slack: configuración/entorno (`channels.slack.*`)
- Listas de permitidos para emparejamiento: `~/.openclaw/credentials/<channel>-allowFrom.json` (cuenta predeterminada) / `<channel>-<accountId>-allowFrom.json` (cuentas no predeterminadas)
- Perfiles de autenticación de modelos: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` (`auth_profile_store`)
- Sesiones OAuth de MCP: `~/.openclaw/state/openclaw.sqlite` (`mcp_oauth_stores`)
- Importación de OAuth heredado: `~/.openclaw/credentials/oauth.json`

Refuerzo de seguridad: mantenga permisos restrictivos (`700` en directorios, `600` en archivos); use cifrado de disco completo en el host del Gateway; si el host es compartido, es preferible usar una cuenta de usuario del sistema operativo dedicada.

### Permisos de archivos

- `~/.openclaw/openclaw.json`: `600` (solo lectura y escritura para el usuario)
- `~/.openclaw`: `700` (solo para el usuario)

`openclaw doctor` puede advertir sobre estos permisos y ofrecer restringirlos.

### Archivos `.env` del espacio de trabajo

OpenClaw carga archivos `.env` locales del espacio de trabajo para agentes y herramientas, pero nunca permite que anulen silenciosamente los controles de ejecución del Gateway:

- Las variables de entorno de credenciales de proveedores se bloquean en los archivos `.env` de espacios de trabajo no confiables; por ejemplo, `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY` y las claves de autenticación de proveedores declaradas por plugins confiables instalados. En su lugar, coloque las credenciales de proveedores en el entorno del proceso del Gateway, `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`), el bloque `env` de la configuración o una importación opcional del shell de inicio de sesión.
- Toda clave que comience por `OPENCLAW_` se bloquea en los archivos `.env` de espacios de trabajo no confiables, lo que reserva todo el espacio de nombres de ejecución para que un futuro control `OPENCLAW_*` adopte una política de denegación predeterminada, en lugar de poder heredarse silenciosamente de contenido `.env` incluido en el repositorio o proporcionado por un atacante.
- La configuración de enrutamiento de endpoints de canales y proveedores también se bloquea en las anulaciones `.env` del espacio de trabajo (por ejemplo, `MATRIX_HOMESERVER`, `MATTERMOST_URL`, `IRC_HOST`, `SYNOLOGY_CHAT_INCOMING_URL`, `AZURE_SPEECH_ENDPOINT` y otras claves que terminan en `_ENDPOINT`), de modo que un espacio de trabajo clonado no pueda redirigir el tráfico de conectores incluidos mediante la configuración de endpoints locales. Esta configuración debe proceder del entorno del proceso del Gateway, del archivo dotenv global de ejecución, de una configuración explícita o de `env.shellEnv`.
- Las variables confiables del entorno del proceso/sistema operativo, el archivo dotenv global de ejecución, la configuración `env` y la importación habilitada del shell de inicio de sesión siguen aplicándose; esto solo restringe la carga de archivos `.env` del espacio de trabajo.

Los archivos `.env` del espacio de trabajo suelen encontrarse junto al código del agente, incluirse por accidente en commits o ser escritos por herramientas; el bloqueo de las credenciales de proveedores impide que un espacio de trabajo clonado sustituya las cuentas de proveedores por otras controladas por un atacante.

### Registros y transcripciones

OpenClaw almacena las transcripciones de las sesiones en el disco, en `~/.openclaw/agents/<agentId>/sessions/*.jsonl`, para mantener la continuidad de las sesiones y permitir la indexación opcional de la memoria; cualquier proceso o usuario con acceso al sistema de archivos puede leerlas. Considere el acceso al disco como el límite de confianza y restrinja los permisos de `~/.openclaw`; para obtener un aislamiento mayor, ejecute los agentes con usuarios del sistema operativo o hosts separados.

Los registros del Gateway pueden incluir resúmenes de herramientas, errores y URL; las transcripciones de las sesiones pueden incluir secretos pegados, contenido de archivos, salida de comandos y enlaces.

- Mantenga activada la censura de registros y transcripciones (`logging.redactSensitive: "tools"`, valor predeterminado).
- Añada patrones personalizados para su entorno mediante `logging.redactPatterns` (tokens, nombres de host, URL internas).
- Al compartir diagnósticos, es preferible usar `openclaw status --all` (se puede pegar y censura los secretos) en lugar de registros sin procesar.
- Elimine las transcripciones de sesiones y los archivos de registro antiguos si no necesita conservarlos durante mucho tiempo.

Detalles: [Registro](/es/gateway/logging)

## Configuración básica segura (copiar y pegar)

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: { mode: "token", token: "your-long-random-token" },
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

Mantiene el Gateway privado, exige el emparejamiento de mensajes directos y evita bots de grupo siempre activos. Para que la ejecución de herramientas también sea más segura, añada un entorno aislado y deniegue las herramientas peligrosas a cualquier agente que no sea propietario (consulte «Perfiles de acceso por agente» más arriba).

### Números separados (WhatsApp, Signal, Telegram)

Para los canales basados en números de teléfono, considere ejecutar el asistente con un número distinto del personal, de modo que las conversaciones personales permanezcan privadas y el número del bot gestione la automatización dentro de sus propios límites.

## Respuesta ante incidentes

### Contención

1. Deténgalo: cierre la aplicación de macOS (si supervisa el Gateway) o finalice el proceso `openclaw gateway`.
2. Cierre la exposición: establezca `gateway.bind: "loopback"` (o deshabilite Tailscale Funnel/Serve) hasta comprender lo ocurrido.
3. Restrinja el acceso: cambie los mensajes directos y grupos de riesgo a `dmPolicy: "disabled"` / exija menciones y elimine todas las entradas `"*"` que permitan el acceso general.

### Rotación (suponga que existe una vulneración si se filtraron secretos)

1. Rote la autenticación del Gateway (`gateway.auth.token` / `OPENCLAW_GATEWAY_PASSWORD`) y reinícielo.
2. Rote los secretos de los clientes remotos (`gateway.remote.token` / `.password`) en todas las máquinas que puedan llamar al Gateway.
3. Rote las credenciales de proveedores/API (credenciales de WhatsApp, tokens de Slack/Discord, claves de modelos/API en `auth-profiles.json` y valores de cargas útiles de secretos cifrados cuando se utilicen).

### Auditoría

1. Compruebe los registros del Gateway con `openclaw logs` (o `openclaw --profile <profile> logs` para un perfil con nombre). La ruta predeterminada es `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; los perfiles con nombre usan `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`, salvo que `logging.file` la anule.
2. Revise las transcripciones pertinentes: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`.
3. Revise los cambios recientes de configuración que puedan haber ampliado el acceso: `gateway.bind`, `gateway.auth`, políticas de mensajes directos/grupos, `tools.elevated` y cambios de plugins.
4. Vuelva a ejecutar `openclaw security audit --deep` y confirme que se hayan resuelto los hallazgos críticos.

### Recopilación de datos para un informe

- Marca de tiempo, sistema operativo del host del Gateway y versión de OpenClaw.
- Las transcripciones de las sesiones y un fragmento breve del final del registro (después de censurarlo).
- Qué envió el atacante y qué hizo el agente.
- Si el Gateway estuvo expuesto más allá de la interfaz de bucle invertido (LAN/Tailscale Funnel/Serve).

## Detección de secretos

El Pipeline de CI ejecuta el hook de pre-commit `detect-private-key` en el repositorio. Si falla, elimine o rote el material de claves incluido en el commit y, a continuación, reproduzca el problema localmente:

```bash
pre-commit run --all-files detect-private-key
```

## Notificación de problemas de seguridad

¿Ha encontrado una vulnerabilidad en OpenClaw? Notifíquela de forma responsable:

1. Correo electrónico: [security@openclaw.ai](mailto:security@openclaw.ai)
2. No la publique hasta que se haya corregido.
3. Se le reconocerá el mérito (salvo que prefiera permanecer en el anonimato).
