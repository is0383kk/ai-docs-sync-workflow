---
read_when:
    - Has terminado la configuración de la inferencia y quieres que OpenClaw configure el resto
    - Necesita inspeccionar o reparar OpenClaw con el agente de configuración local
    - Está diseñando o habilitando el modo de rescate del canal de mensajes
summary: Referencia de la CLI y modelo de seguridad del asistente de configuración y reparación de OpenClaw basado en inferencia
title: Agente de configuración de OpenClaw
x-i18n:
    generated_at: "2026-07-26T05:03:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw incluye un agente del sistema integrado —se presenta como "OpenClaw"— para
la configuración, reparación y administración locales (anteriormente denominado Crestodian). Solo se inicia después de que el modelo predeterminado efectivo complete un turno real.
Las instalaciones nuevas establecen primero la inferencia; las configuraciones incorrectas permanecen en la
ruta clásica de doctor.

## Cuándo se inicia

Al ejecutar `openclaw` sin ningún subcomando, se determina la ruta según el estado de la configuración:

- Falta la configuración o existe sin ajustes definidos (vacía o solo con las claves `$schema`/`meta`): inicia la incorporación guiada con verificación de IA en vivo.
- La configuración existe, pero no supera la validación: inicia la incorporación clásica, que informa de los problemas y remite a `openclaw doctor`.
- La configuración existe y es válida: abre la TUI normal del agente. Un Gateway accesible
  y configurado cuyo agente predeterminado tenga un modelo accede directamente a esa interfaz
  sin pasar por la incorporación ni por OpenClaw. Use `/openclaw` dentro de la TUI o ejecute
  directamente `openclaw setup` para acceder a OpenClaw más adelante.

Al ejecutar `openclaw setup`, primero se prueba en vivo el modelo predeterminado configurado. Un turno correcto inicia OpenClaw. Un fallo interactivo abre la configuración guiada de la inferencia y transfiere el control a OpenClaw después de que un candidato supere la prueba. Las solicitudes de una sola ejecución, JSON y otras solicitudes no interactivas fallan con instrucciones para ejecutar `openclaw onboard` cuando la inferencia no está disponible. `openclaw --help` y `openclaw --version` conservan sus rutas rápidas habituales.

La ejecución no interactiva de `openclaw` sin argumentos (sin TTY) finaliza con un mensaje breve en lugar de mostrar la ayuda raíz: remite a la incorporación no interactiva en una instalación nueva o no válida, o a `openclaw agent --local ...` cuando la configuración es válida.

`openclaw onboard --modern` sigue siendo un alias de compatibilidad de OpenClaw, pero utiliza la misma puerta de inferencia: si la inferencia funciona, abre el chat; los fallos interactivos inician la configuración guiada de la inferencia y los fallos no interactivos finalizan con indicaciones sobre la incorporación. `openclaw onboard --classic` abre el asistente completo paso a paso.

## Qué muestra OpenClaw

OpenClaw interactivo abre el mismo entorno de TUI que `openclaw tui`, con un backend de chat de OpenClaw. El saludo inicial abarca:

- la validez de la configuración y el agente predeterminado
- el modelo verificado que utiliza OpenClaw
- la accesibilidad del Gateway según la primera comprobación de inicio
- la siguiente acción de depuración recomendada

No vuelca secretos ni carga comandos de la CLI de los plugins solo para iniciarse.

Use `status` para consultar el inventario detallado: ruta de configuración, rutas de la documentación y del código fuente, comprobaciones de la CLI local, presencia de claves o tokens, agentes, modelo y detalles del Gateway.

OpenClaw utiliza el mismo descubrimiento de referencias que los agentes normales: en un repositorio Git, remite al `docs/` local y al árbol de código fuente; en una instalación de npm, utiliza la documentación incluida y enlaza con [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw), con la recomendación de consultar el código fuente cuando la documentación no sea suficiente.

## Ejemplos

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "models"
openclaw setup --message "validate config"
openclaw setup --message "setup workspace ~/Projects/work" --yes
openclaw setup --message "set default model openai/gpt-5.6" --yes
openclaw onboard --modern
```

Dentro de la TUI de OpenClaw:

```text
status
health
doctor
validate config
setup
setup workspace ~/Projects/work
config set gateway.port 19001
config set-ref gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
gateway status
restart gateway
agents
create agent work workspace ~/Projects/work
models
configure model provider
set default model openai/gpt-5.6
channels
channel info slack
connect slack
open channel wizard for slack
plugins list
plugins search slack
plugin install clawhub:openclaw-codex-app-server
talk to work agent
talk to agent for ~/Projects/work
audit
quit
```

## Operaciones y aprobación

OpenClaw utiliza operaciones tipadas en lugar de editar la configuración de forma improvisada.

Las operaciones de solo lectura se ejecutan inmediatamente: mostrar el resumen, enumerar los agentes, enumerar los plugins instalados, buscar plugins de ClawHub, mostrar el estado del modelo o del backend, ejecutar comprobaciones de estado o funcionamiento, comprobar la accesibilidad del Gateway, ejecutar doctor sin correcciones interactivas, validar la configuración y mostrar la ruta del registro de auditoría.

El inicio de la configuración guiada del canal (`connect telegram`) también se ejecuta inmediatamente. Su asistente recopila respuestas explícitas y controla las escrituras resultantes.

Las operaciones persistentes requieren aprobación conversacional (o `--yes` para un comando directo): escribir la configuración, `config set`, `config set-ref`, inicializar la configuración o la incorporación, cambiar el modelo predeterminado, iniciar, detener o reiniciar el Gateway, crear agentes e instalar plugins.

Las reparaciones de doctor no están disponibles dentro de OpenClaw porque pueden reescribir el proveedor, la autenticación o la ruta de inferencia del agente predeterminado que sustenta la sesión. Salga de OpenClaw y ejecute `openclaw doctor --fix` en una terminal. La versión de solo lectura de `doctor` sigue estando disponible dentro de OpenClaw.

Los agentes nuevos heredan la ruta de inferencia predeterminada verificada en vivo. Los identificadores de agente `openclaw` y `crestodian` están reservados para el agente del sistema y no pueden crearse como agentes normales. El identificador retirado permanece bloqueado para que una configuración antigua no pueda reclamarlo.

`config set` y `config set-ref` pueden cambiar cualquier ajuste que pueda cambiar un usuario,
con una breve lista de denegación destinada únicamente a personas: `$include`, `auth.*`, `env.*`, `models.*`
y `secrets.*` se siguen rechazando porque contienen material de credenciales,
inclusión de configuraciones alternativas o las definiciones de proveedores o catálogos que alimentan
el enrutamiento de inferencia. El propio enrutamiento de inferencia también está protegido: se rechazan las rutas
del modelo predeterminado (los campos de modelo, parámetros y entorno de ejecución de `agents.defaults`) y los campos de enrutamiento
del agente que respalda la ruta predeterminada activa, al igual que los campos
de identidad o topología del agente (`id`, `agentDir`, `default`). Los campos de enrutamiento de
otros agentes pueden escribirse con aprobación. La autenticación del Gateway y de los canales sigue siendo
una superficie de configuración normal. Use `set default model <provider/model>` para una
ruta ya configurada; se prueba la ruta en vivo antes de guardarla. Para
configurar o reparar el acceso al proveedor o a la autenticación, salga de OpenClaw y ejecute
`openclaw onboard`.

Se permiten las escrituras de `plugins.entries.<id>.*` (habilitar, deshabilitar o configurar plugins instalados),
salvo que ese plugin respalde la ruta de inferencia activa. Las fuentes de
instalación de plugins y la política de carga mantienen su límite de confianza en el flujo de trabajo tipado
de instalación de plugins. La desinstalación del plugin que respalda la ruta
se rechaza por el mismo motivo; salga de OpenClaw y ejecute
`openclaw plugins uninstall <id>` desde una terminal.

La aprobación se expresa con sus propias palabras: las respuestas inequívocas ("sí", "claro", "adelante", "ahora no") se resuelven mediante una lista cerrada y determinista. Cuando la ruta configurada admite una llamada de finalización independiente, otras respuestas pueden clasificarse utilizando únicamente su mensaje y la propuesta pendiente, nunca mediante el propio modelo de conversación, que no puede autoaprobarse. Las respuestas no clasificadas o ambiguas mantienen pendiente la propuesta y se vuelve a preguntar durante la conversación.

### Historial de cambios

La página Preguntar a OpenClaw puede mostrar las operaciones recientes aplicadas por el agente del sistema, las migraciones de Doctor,
las escrituras de configuración de Ajustes y de la CLI, y las modificaciones manuales de
`openclaw.json`. El diario de configuración detecta las modificaciones externas mientras el Gateway
está supervisando, durante una escritura controlada por OpenClaw o en el siguiente inicio después de una
modificación sin conexión.

El historial se almacena en la tabla `diagnostic_events` de la base de datos
compartida `~/.openclaw/state/openclaw.sqlite`, dentro de los ámbitos `system-agent-audit`
y `config-audit`. Cada ámbito conserva sus 50,000 registros más recientes.
No se incluyen las operaciones de descubrimiento ni de solo lectura. Los secretos nunca aparecen en
el historial de cambios; los registros del diario de configuración contienen las rutas modificadas en lugar de los valores
de configuración, y la comparación de valores utiliza huellas digitales protegidas.

La configuración de canales puede ejecutarse como una conversación alojada hasta que se necesita un secreto. La
TUI local de OpenClaw no acepta respuestas confidenciales del asistente porque la entrada del
chat de la terminal es visible. Ofrece inmediatamente `open channel wizard`, que transfiere
el canal seleccionado al asistente enmascarado de la terminal; también puede ejecutar
`openclaw channels add --channel <channel>` más adelante.

### Cambio a la configuración enmascarada de canales

El chat local puede transferir el control al asistente enmascarado de canales:

```text
open channel wizard for slack
channel info slack
```

`open channel wizard for <channel>` abre la configuración enmascarada del canal después de que se cierre la
TUI del chat. Use primero `channel info <channel>` para consultar la etiqueta del canal, el estado de
configuración, el resumen de requisitos previos y el enlace a la documentación.

OpenClaw nunca cambia el acceso al proveedor o a la autenticación desde su propia sesión: la
sesión ya depende de esa ruta de inferencia. Para configurar o
reparar el proveedor del modelo, `configure model provider` devuelve indicaciones para salir y realizar la incorporación sin
iniciar un asistente ni escribir la configuración. Salga de OpenClaw y ejecute `openclaw
onboard`; la incorporación prepara las credenciales y guarda únicamente una ruta que
complete un turno real en vivo. Vuelva a iniciar OpenClaw después de que la incorporación se complete correctamente.

## Inicialización de la configuración

`setup` configura el resto del espacio de trabajo y el estado del Gateway después de que la incorporación guiada ya haya establecido la inferencia. Solo escribe mediante operaciones de configuración tipadas y solicita aprobación primero.

```text
setup
setup workspace ~/Projects/work
```

`setup` conserva el modelo efectivo verificado. No configura ni
reemplaza la inferencia.

Si falta la inferencia o falla su comprobación en vivo, salga de OpenClaw y ejecute `openclaw onboard`. La incorporación guiada prueba primero el modelo configurado y, a continuación, las CLI de suscripciones autenticadas, las claves de API y las demás CLI compatibles; solicita una respuesta real a cada candidato y conserva únicamente una ruta que supere la prueba. OpenClaw se inicia inmediatamente después de ese límite y puede configurar entonces el espacio de trabajo, el Gateway, los canales, los agentes, los plugins y otras funciones opcionales.

La aplicación de macOS omite por completo esta secuencia cuando accede a un Gateway configurado
cuyo agente predeterminado ya tiene un modelo configurado; abre la interfaz normal del
agente.
Para un Gateway nuevo o incompleto, la aplicación gestiona la secuencia de inferencia mediante
los métodos del Gateway `openclaw.setup.detect` y `openclaw.setup.activate`:
la detección enumera todos los backends candidatos que encuentra; la activación prueba en vivo un
candidato (una finalización real de "responde con OK") y solo conserva el modelo,
la credencial y el estado del proveedor o del entorno de ejecución necesarios para esa ruta después de que la prueba se complete correctamente. Los valores predeterminados del espacio de trabajo y del Gateway quedan a cargo de OpenClaw. Un candidato que falla
nunca modifica la configuración; la aplicación avanza automáticamente por la secuencia y, finalmente,
ofrece un paso manual de clave o token rellenado con los plugins de proveedores
de inferencia de texto activos del Gateway. El proveedor seleccionado controla su modelo
inicial y su configuración, y la credencial se verifica de la misma forma antes de guardarse.

La supervisión de Codex y otras funciones opcionales de plugins permanecen fuera de esta
transacción de activación de la inferencia. Configúrelas únicamente después de que la inferencia
funcione y OpenClaw se haya iniciado; la política de plugins existente y las
exclusiones explícitas de supervisión no se modifican durante la configuración de la inferencia.

## Conversación con IA

La conversación libre de OpenClaw interactivo se ejecuta mediante el mismo bucle de agente que los agentes normales de OpenClaw, restringido a una herramienta de autoridad de nivel cero de OpenClaw, `openclaw`, que encapsula las operaciones tipadas. Las acciones de lectura se ejecutan libremente; las modificaciones requieren su aprobación conversacional para esa operación exacta (consulte Operaciones y aprobación), y cada escritura aplicada se audita y se vuelve a validar. La sesión del agente persiste, por lo que OpenClaw dispone de memoria real entre varios turnos. Si la ruta de inferencia verificada deja de funcionar más adelante, vuelva a `openclaw onboard` y repárela antes de continuar.

El host no analiza solicitudes en lenguaje natural para convertirlas en operaciones. Los mensajes de formato libre
—incluido el texto que parece un comando y preguntas como "¿por qué se detuvo mi
gateway?"— se envían a la IA, que puede asignar la solicitud a una operación tipada
mediante la herramienta `openclaw`.

Cuando hay una mutación pendiente, solo se resuelven sin inferencia las frases inequívocas de aprobación o rechazo incluidas en una
lista cerrada. El consentimiento ambiguo se envía a una
llamada de finalización configurada por separado y, de lo contrario, falla de forma cerrada. Los campos estructurados
del asistente y la navegación exacta del host son controles de la interfaz de usuario, no análisis de operaciones
en lenguaje natural. Una excepción de higiene de secretos es especialmente importante: un
`config set` exacto en una ruta sensible (tokens, claves, contraseñas) nunca llega
a un modelo. El host crea una propuesta censurada y el valor se enmascara en el
historial visible para la IA. Se recomienda `config set-ref <path> env <ENV_VAR>` para los secretos.

El modo de rescate del canal de mensajes nunca utiliza el planificador asistido por modelos. El rescate remoto sigue siendo determinista para que una ruta normal de agente averiada o comprometida no pueda utilizarse como editor de configuración.

### Modelo de confianza del arnés de la CLI

Los entornos de ejecución integrados y el arnés del servidor de aplicaciones de Codex aplican directamente la restricción
de nivel cero: la ejecución lleva una lista de herramientas permitidas de OpenClaw que contiene únicamente
la herramienta `openclaw`. Para Codex, OpenClaw también deshabilita los entornos, la ejecución
nativa, los múltiples agentes, los objetivos, las aplicaciones/plugins, las Skills/MCP, la búsqueda web y
las superficies `request_user_input` para esa ejecución. Codex sigue inyectando su utilidad nativa inerte `update_plan`;
esta puede actualizar la lista de comprobación temporal del modelo, pero no puede escribir archivos
ni la configuración de OpenClaw. Los arneses de la CLI no consumen la lista de elementos permitidos de OpenClaw,
por lo que OpenClaw solo admite backends cuyo propio contrato de selección de herramientas pueda demostrar
la misma restricción:

- Los backends seleccionables, incluido Claude Code, se inician con una selección vacía de herramientas
  nativas y una herramienta MCP, `openclaw`. La configuración MCP generada de Claude se
  aplica con `--strict-mcp-config`, por lo que no se carga ningún otro servidor MCP.
- Los backends que declaran no tener herramientas nativas reciben el mismo servidor MCP dedicado
  de OpenClaw.
- Los backends con herramientas nativas siempre activas o desconocidas fallan de forma cerrada antes de la inferencia;
  no pueden alojar una sesión de OpenClaw.

Solo las sesiones de OpenClaw reciben el servidor MCP de openclaw; las ejecuciones normales de agentes
nunca ven esta herramienta. Por tanto, los backends de CLI seleccionables/sin herramientas nativas y los modelos
con clave de API aplican el bucle literal de una sola herramienta. Los modelos del servidor de aplicaciones de Codex aplican
una única herramienta de autoridad de OpenClaw más la utilidad de planificación nativa inerte. En los
tres casos, las escrituras de configuración permanecen confinadas al contrato de aprobación auditado
de OpenClaw.

Gemini CLI sigue disponible para los agentes normales, pero no puede aplicar la
prueba sin herramientas exigida por la puerta de inferencia, por lo que no puede alojar OpenClaw.

## Cambio a un agente

Utilice un selector en lenguaje natural para salir de OpenClaw y abrir la TUI normal:

```text
hablar con el agente
hablar con el agente de trabajo
cambiar al agente principal
```

`openclaw tui`, `openclaw chat` y `openclaw terminal` abren directamente la TUI del agente normal; no inician OpenClaw. Tras cambiar a la TUI normal, `/openclaw` vuelve a OpenClaw, opcionalmente con una solicitud de seguimiento:

```text
/openclaw
/openclaw reiniciar gateway
```

## Modo de rescate de mensajes

El modo de rescate de mensajes es el punto de entrada de OpenClaw desde el canal de mensajes: utilícelo cuando el agente normal no funcione pero un canal de confianza (por ejemplo, WhatsApp) siga recibiendo comandos.

Se trata de un controlador determinista de comandos de emergencia, no del agente
conversacional de OpenClaw. No inicializa una configuración nueva ni relaja la puerta de
inferencia para el chat de OpenClaw.

Comando compatible: `/openclaw <request>`. El rescate solo acepta la gramática exacta de comandos escritos: el lenguaje natural se rechaza con una indicación, nunca se interpreta de forma aproximada como una operación y nunca se consulta ningún modelo.

```text
Usted, en un MD de propietario de confianza: /openclaw status
OpenClaw: Modo de rescate de OpenClaw. Gateway accesible: no. Configuración válida: no.
Usted: /openclaw restart gateway
OpenClaw: Plan: reiniciar el Gateway. Responda /openclaw yes para aplicarlo.
Usted: /openclaw yes
OpenClaw: Aplicado. Entrada de auditoría escrita.
```

La creación de agentes también puede ponerse en cola localmente o mediante el rescate:

```text
create agent work workspace ~/Projects/work model openai/gpt-5.6-sol
/openclaw create agent work workspace ~/Projects/work
```

La creación de agentes solo puede indicar el modelo predeterminado actual verificado en vivo. Omita el
modelo para heredar esa ruta.

El rescate remoto es una superficie de administración y debe tratarse como una reparación remota de la configuración, no como un chat normal.

Contrato de seguridad para el rescate remoto:

- Se deshabilita cuando el aislamiento está activo para el agente o la sesión; OpenClaw rechaza el rescate remoto e indica que se utilice la reparación local mediante la CLI.
- El estado efectivo predeterminado es `auto`: permitir el rescate remoto únicamente en operaciones YOLO de confianza, donde el entorno de ejecución ya tiene autoridad local sin aislamiento (`tools.exec.security` se resuelve como `full` y `tools.exec.ask` se resuelve como `off`, con el modo de aislamiento `off`).
- Requiere una identidad de propietario explícita; no se permiten reglas de remitente comodín, políticas de grupo abiertas, webhooks no autenticados ni canales anónimos.
- El rescate se limita a los MD del propietario.
- La búsqueda y la lista de plugins son de solo lectura. La instalación de plugins siempre es exclusivamente local (está bloqueada en el rescate, incluso cuando está habilitada de otro modo), ya que descarga código ejecutable. La desinstalación de plugins se rechaza tanto en OpenClaw local como en el rescate; ejecute `openclaw plugins uninstall <id>` desde un terminal.
- El rescate remoto no puede abrir la TUI local ni cambiar a una sesión interactiva de agente; utilice `openclaw` localmente para transferir el control al agente.
- Las escrituras persistentes siguen requiriendo aprobación, incluso en el modo de rescate.
- Las aprobaciones pendientes son de un solo uso. Cualquier comando de rescate posterior para la misma cuenta, canal y remitente revoca el plan anterior; una ejecución fallida también consume la aprobación, por lo que debe volver a enviar el comando para reintentarlo.
- Cada operación de rescate aplicada se audita. El rescate mediante canales de mensajes registra los metadatos del canal, la cuenta, el remitente y la dirección de origen; las operaciones que modifican la configuración también registran los hashes de configuración anteriores y posteriores.
- Los secretos nunca se reproducen. La inspección de SecretRef informa sobre la disponibilidad, no sobre los valores.
- Si el Gateway está activo, el rescate prioriza las operaciones tipadas del Gateway; si no está activo, el rescate utiliza únicamente la superficie mínima de reparación local que no depende del bucle normal del agente.

La política de rescate está integrada: solo está disponible cuando el entorno de ejecución efectivo es
YOLO, el aislamiento está desactivado y la solicitud es un MD del propietario. Las aprobaciones de escritura pendientes
caducan después de 15 minutos. `openclaw doctor --fix` elimina los bloques de configuración retirados
`systemAgent` y `crestodian`.

El rescate remoto está cubierto por la vía de Docker:

```bash
pnpm test:docker:system-agent-rescue
```

Una prueba de humo opcional de la superficie de comandos del canal en vivo comprueba `/openclaw status` junto con un ciclo completo de aprobación persistente mediante el controlador de rescate:

```bash
pnpm test:live:system-agent-rescue-channel
```

La configuración empaquetada de una sola ejecución controlada por inferencia está cubierta por:

```bash
pnpm test:docker:system-agent-first-run
```

Esa vía de la CLI empaquetada comienza con un directorio de estado vacío y demuestra que OpenClaw
falla de forma cerrada sin inferencia. Después prueba y activa una instancia falsa de Claude mediante
el módulo de activación empaquetado. Solo entonces una solicitud imprecisa llega al
planificador y se resuelve como una configuración tipada, seguida de comandos de una sola ejecución que crean un
agente adicional, configuran Discord mediante la habilitación de un plugin junto con un SecretRef
de token, validan la configuración y comprueban el registro de auditoría. Esta vía aporta
evidencia sobre la puerta y las operaciones; no ejercita la incorporación interactiva ni la
conversación entre el agente, las herramientas y las aprobaciones de OpenClaw. El escenario de QA Lab siguiente
redirige a la misma vía de Docker:

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Doctor](/es/cli/doctor)
- [TUI](/es/cli/tui)
- [Aislamiento](/es/cli/sandbox)
- [Seguridad](/es/cli/security)
