---
read_when:
    - Uso o modificación de la herramienta exec
    - Depuración del comportamiento de stdin o TTY
summary: Uso de la herramienta exec, modos de stdin y compatibilidad con TTY
title: Herramienta de ejecución
x-i18n:
    generated_at: "2026-07-26T05:00:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

Ejecuta comandos de shell en el espacio de trabajo. `exec` es una superficie de shell con capacidad de modificación: los comandos pueden crear, editar o eliminar archivos dondequiera que lo permita el sistema de archivos del host o sandbox seleccionado. Deshabilitar herramientas del sistema de archivos de OpenClaw como `write`, `edit` o `apply_patch` no convierte `exec` en solo lectura.

Admite la ejecución en primer plano y en segundo plano mediante `process`. Si `process` no está permitido, `exec` se ejecuta de forma síncrona e ignora `yieldMs`/`background`. Las sesiones en segundo plano tienen un ámbito por agente; `process` solo ve las sesiones del mismo agente.

## Parámetros

<ParamField path="command" type="string" required>
Comando de shell que se ejecutará.
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
Directorio de trabajo del comando.
</ParamField>

<ParamField path="env" type="object">
Sobrescrituras de entorno de clave/valor que se combinan con el entorno heredado.
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
Envía automáticamente el comando a segundo plano después de este retraso (ms).
</ParamField>

<ParamField path="background" type="boolean" default="false">
Envía el comando a segundo plano inmediatamente en lugar de esperar a `yieldMs`.
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
Sobrescribe el tiempo de espera de ejecución configurado para esta llamada, en segundos. Se aplica a la ejecución en primer plano, en segundo plano, de `yieldMs`, del gateway, del sandbox y de `system.run` del Node. `timeout: 0` deshabilita el tiempo de espera del proceso de ejecución para esa llamada.
</ParamField>

<ParamField path="pty" type="boolean" default="false">
Ejecuta en un pseudoterminal cuando esté disponible. Se utiliza para las CLI que solo funcionan con TTY, los agentes de programación y las interfaces de terminal.
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
Dónde ejecutar. `auto` se resuelve como `sandbox` cuando hay un entorno de ejecución de sandbox activo y como `gateway` en caso contrario.
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
Se ignora para las llamadas normales a herramientas. La seguridad de `gateway`/`node` se deriva de `tools.exec.mode` y del archivo de aprobaciones del host; el modo elevado solo puede forzar el acceso completo cuando el operador concede explícitamente el acceso elevado.
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
El modo de solicitud de referencia se deriva de `tools.exec.mode` y de las aprobaciones del host. Para las llamadas al modelo originadas en un canal, `ask` por llamada se ignora cuando la solicitud efectiva del host es `off`; de lo contrario, solo puede reforzarse a un modo más estricto.
</ParamField>

<ParamField path="node" type="string">
Id/nombre del Node cuando `host=node`.
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
Solicita el modo elevado: sale del sandbox hacia la ruta configurada del host. `security=full` solo se fuerza cuando el modo elevado se resuelve como `full`.
</ParamField>

Notas:

- `host` solo acepta `auto`, `sandbox`, `gateway` o `node`. No es un selector de nombre de host; los valores con aspecto de nombre de host se rechazan antes de ejecutar el comando.
- `host=node` por llamada se permite desde `auto`; `host=gateway` por llamada solo se permite cuando no hay ningún entorno de ejecución de sandbox activo.
- Sin configuración adicional, `host=auto` sigue «funcionando sin más»: si no hay sandbox, se resuelve como `gateway`; si hay un sandbox activo, permanece en el sandbox.
- `elevated` sale del sandbox hacia la ruta configurada del host: `gateway` de forma predeterminada, o `node` cuando `tools.exec.host=node` (o el valor predeterminado de la sesión es `host=node`). Solo está disponible cuando el acceso elevado está habilitado para la sesión o el proveedor actuales.
- Las aprobaciones de `gateway`/`node` están controladas por el archivo de aprobaciones del host.
- `node` requiere un Node emparejado (una aplicación complementaria o un host de Node sin interfaz gráfica). Si hay varios Nodes disponibles, establece `exec.node` o `tools.exec.node` para seleccionar uno.
- `exec host=node` es la única ruta de ejecución de shell para los Nodes; se ha eliminado el contenedor heredado `nodes.run`.
- En hosts que no sean Windows, la ejecución utiliza `SHELL` cuando está establecido; si `SHELL` es `fish`, prefiere `bash` (o `sh`) de `PATH` para evitar construcciones de Bash incompatibles con fish y, si ninguno existe, recurre a `SHELL`.
- En hosts Windows, la ejecución prefiere detectar PowerShell 7 (`pwsh`) (Program Files, ProgramW6432 y después PATH) y, si no está disponible, recurre a Windows PowerShell 5.1.
- En hosts de Gateway que no sean Windows, los comandos de ejecución de Bash y zsh utilizan una instantánea de inicio. OpenClaw captura los alias y funciones que se pueden cargar, así como un pequeño conjunto seguro de variables de entorno, de los archivos de inicio del shell en `$OPENCLAW_STATE_DIR/cache/shell-snapshots/`, y después carga esa instantánea antes de cada comando de ejecución. Se excluyen las variables que parecen contener secretos; la ejecución en sandbox y Node no utiliza esta instantánea. Establece `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` en el entorno del proceso del Gateway para deshabilitar esta ruta de instantánea.
- La ejecución en el host (`gateway`/`node`) rechaza `env.PATH` y las sobrescrituras del cargador (`LD_*`/`DYLD_*`) para impedir el secuestro de binarios o la inyección de código.
- OpenClaw establece `OPENCLAW_SHELL=exec` en el entorno del comando generado (incluidas las ejecuciones con PTY y en sandbox) para que las reglas del shell o del perfil puedan detectar el contexto de la herramienta de ejecución.
- Para las ejecuciones originadas en un canal, OpenClaw también expone una carga JSON limitada con la identidad del remitente/chat en `OPENCLAW_CHANNEL_CONTEXT` cuando el canal proporciona esos identificadores.
- `exec` no puede ejecutar los comandos de shell `openclaw channels login` ni `/approve`: `openclaw channels login` es un flujo interactivo de autenticación de canal y `/approve` debe pasar por el controlador de comandos de aprobación, no por un shell. Ejecuta el inicio de sesión del canal en un terminal del host del Gateway o utiliza una herramienta de agente de inicio de sesión específica del canal cuando exista (por ejemplo, `whatsapp_login`).
- Importante: el aislamiento en sandbox está **desactivado de forma predeterminada**. Si está desactivado, `host=auto` implícito se resuelve como `gateway`. `host=sandbox` explícito sigue fallando de forma segura en lugar de ejecutarse silenciosamente en el host del Gateway. Habilita el aislamiento en sandbox o utiliza `host=gateway` con aprobaciones.
- Las comprobaciones previas de scripts (para detectar errores comunes de sintaxis de shell en Python/Node) solo inspeccionan archivos dentro del límite efectivo de `workdir`. Si la ruta de un script se resuelve fuera de `workdir`, se omite la comprobación previa de ese archivo. La comprobación previa también se omite por completo cuando `host=gateway` y la política efectiva es `security=full` con `ask=off`.
- Para trabajos de larga duración que comienzan ahora, inícialos una sola vez y utiliza la reactivación automática al completarse cuando esté habilitada y el comando produzca salida o falle. Utiliza `process` para consultar registros y estado, proporcionar entradas o intervenir; no emules la programación mediante bucles de suspensión, bucles de tiempo de espera ni sondeos repetidos.
- Los comandos en segundo plano iniciados por el agente aparecen en las vistas de tareas en segundo plano de la Web, iOS y Android hasta que finalizan. El registro de tareas se finaliza antes de que el Heartbeat de finalización vuelva a activar al agente.
- Para trabajos que deban realizarse más adelante o según una programación, utiliza Cron en lugar de los patrones de suspensión/retraso de `exec`.

## Configuración

| Clave                                  | Valor predeterminado                  | Notas                                                                                                                                                   |
| ------------------------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | Tiempo de espera predeterminado por comando de ejecución, en segundos. `timeout` por llamada lo sobrescribe; `timeout: 0` por llamada deshabilita el tiempo de espera del proceso de ejecución.                  |
| `tools.exec.host`                    | `auto`                   | Se resuelve como `sandbox` cuando hay un entorno de ejecución de sandbox activo y como `gateway` en caso contrario.                                                                            |
| `tools.exec.mode`                    | derivado del host             | Opción canónica de la política. Consulta [Modos](#modes) a continuación.                                                                                                       |
| `tools.exec.reviewer.model`          | proveedor/modelo principal configurado del agente | Sobrescritura opcional del proveedor/modelo para la revisión de `mode=auto`.                                                                                                |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | Tiempo de espera por etapa para la preparación y finalización del modelo revisor antes de recurrir a una persona.                                                                  |
| `tools.exec.node`                    | sin establecer                    |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | Cuando es verdadero, las sesiones de ejecución enviadas a segundo plano ponen en cola un evento del sistema y solicitan un Heartbeat al finalizar.                                                           |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | Emite un único aviso de «en ejecución» cuando una ejecución sujeta a aprobación tarda más que este valor (`0` lo deshabilita).                                                        |
| `tools.exec.strictInlineEval`        | `false`                  | Consulta [Evaluación en línea](#inline-eval-strictinlineeval).                                                                                                       |
| `tools.exec.commandHighlighting`     | `false`                  | Cuando es verdadero, las solicitudes de aprobación pueden resaltar en el texto del comando los segmentos de comando derivados del analizador. Se establece globalmente o por agente; no modifica la política de aprobación. |
| `tools.exec.pathPrepend`             | sin establecer                    | Lista de directorios que se antepondrán a `PATH` para las ejecuciones (solo Gateway y sandbox).                                                                        |
| `tools.exec.safeBins`                | sin establecer                    | Binarios seguros que solo leen de stdin y pueden ejecutarse sin entradas explícitas en la lista de permitidos. Consulta [Binarios seguros](/es/tools/exec-approvals-advanced#safe-bins-stdin-only).         |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | Directorios explícitos adicionales de confianza para las comprobaciones de rutas de `safeBins`. Las entradas de `PATH` nunca se consideran de confianza automáticamente.                                              |
| `tools.exec.safeBinProfiles`         | sin establecer                    | Política argv personalizada opcional por binario seguro (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).                                        |

La ejecución en el host sin aprobación es el valor predeterminado para el Gateway y el Node (`mode=full`); esto procede de los valores predeterminados de la política del host, no de `host=auto`. Si se desea un comportamiento de aprobaciones/lista de permitidos, establece `tools.exec.mode` y restringe el archivo de aprobaciones del host; consulta [Aprobaciones de ejecución](/es/tools/exec-approvals#yolo-mode-no-approval). Para forzar el enrutamiento al Gateway o al Node independientemente del estado del sandbox, establece `tools.exec.host` o utiliza `/exec host=...`.

Ejemplo:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Modos

`tools.exec.mode` es la opción canónica de política persistente. El comportamiento de seguridad y aprobación en tiempo de ejecución se deriva de ella.

| Modo        | seguridad    | solicitud       | Comportamiento                                                                                                                       |
| ----------- | ----------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `deny`      | `deny`      | `off`     | Se deniega la ejecución.                                                                                                                |
| `allowlist` | `allowlist` | `off`     | Solo se ejecutan los comandos incluidos en la lista de permitidos o los binarios seguros; no se solicita nada más.                                                                 |
| `ask`       | `allowlist` | `on-miss` | Las coincidencias con la lista de permitidos se ejecutan directamente; todo lo demás requiere la aprobación de una persona.                                                                  |
| `auto`      | `allowlist` | `on-miss` | Las coincidencias con la lista de permitidos o los binarios seguros se ejecutan directamente; todo lo demás pasa por el revisor automático nativo de OpenClaw antes de solicitar la aprobación de una persona. |
| `full`      | `full`      | `off`     | Sin puerta de aprobación.                                                                                                              |

La opción por sesión `/exec ask=always` sigue solicitando la aprobación de una persona cada vez, independientemente del modo persistente.

La aprobación mediante revisión automática es de un solo uso. En el Gateway, OpenClaw proporciona al revisor la ruta resuelta del ejecutable y fija la ejecución a esa misma ruta. Los comandos que no puedan reducirse a un único plan de ejecución aplicable —como heredocs, expansiones del shell o entrecomillado no compatible de envoltorios— recurren a la aprobación humana aunque el modelo los permitiera en otras circunstancias.

Las aprobaciones de comandos del servidor de aplicaciones de Codex que no estén ya decididas por una política explícita del entorno de ejecución o una política nativa utilizan la vía de aprobación humana. OpenClaw no ejecuta su revisor de ejecución configurado para estas solicitudes porque Codex no expone un ejecutable resuelto aplicable que permita vincular la decisión de revisión al comando que ejecuta Codex.

### Evaluación en línea (`strictInlineEval`)

Cuando `tools.exec.strictInlineEval` es `true`, las formas de evaluación en línea del intérprete requieren revisión o aprobación explícita: `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, `osascript -e` y formas similares en otros intérpretes y portadores de comandos compatibles (`awk`, `find -exec`, `make`, `sed`, `xargs` y más). En `mode=auto`, la vía normal de aprobación de ejecución puede permitir que el revisor automático nativo autorice un comando puntual que claramente presente poco riesgo; las llamadas directas `system.run` al host del nodo siguen requiriendo aprobación explícita porque no pueden transferir el comando a una vía de aprobación humana. Si el revisor lo solicita, la solicitud se envía a una persona. `allow-always` puede seguir conservando invocaciones benignas de intérpretes o scripts, pero las formas de evaluación en línea no se convierten en reglas de permiso permanentes.

### Gestión de PATH

- `host=gateway`: combina el `PATH` del shell de inicio de sesión con el entorno de ejecución. Se rechazan las sobrescrituras de `env.PATH` para la ejecución en el host. El propio daemon continúa ejecutándose con un `PATH` mínimo:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
  - Para impedir que la configuración del shell del usuario (como `~/.zshenv` o `/etc/zshenv`) sobrescriba las rutas prioritarias durante el inicio, las entradas de `tools.exec.pathPrepend` se anteponen de forma segura al `PATH` final dentro del comando del shell justo antes de la ejecución.
- `host=sandbox`: ejecuta `sh -lc` (shell de inicio de sesión) dentro del contenedor, por lo que `/etc/profile` puede restablecer `PATH`. OpenClaw antepone `env.PATH` después de cargar el perfil mediante una variable de entorno interna (sin interpolación del shell); `tools.exec.pathPrepend` también se aplica aquí.
- `host=node`: solo se envían al nodo las sobrescrituras de entorno no bloqueadas que se proporcionen. Se rechazan las sobrescrituras de `env.PATH` para la ejecución en el host y los hosts de nodos las ignoran. Si se necesitan entradas adicionales de PATH en un nodo, configure el entorno del servicio del host del nodo (systemd/launchd) o instale las herramientas en ubicaciones estándar.

Vinculación de nodo por agente (use en la configuración el identificador de agente utilizado como clave):

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Interfaz de control: la página **Dispositivos** incluye un pequeño panel «Vinculación del nodo de ejecución» para la misma configuración.

## Sobrescrituras de sesión (`/exec`)

Use `/exec` para establecer los valores predeterminados **por sesión** de `host`, `security`, `ask` y `node`. Envíe `/exec` sin argumentos para mostrar los valores actuales.

Ejemplo:

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

`/exec` solo se admite para **remitentes autorizados** mediante listas de permitidos o emparejamiento del canal y grupos de acceso. La aplicación de los grupos de acceso está siempre activa. Solo actualiza el **estado de la sesión** y no escribe la configuración. Los remitentes autorizados de canales externos pueden establecer estos valores predeterminados de sesión. Los clientes internos del Gateway o del chat web necesitan `operator.admin` para conservarlos.

Para desactivar por completo la ejecución, deniéguela mediante la política de herramientas (`tools.deny: ["exec"]` o por agente). Las aprobaciones del host siguen aplicándose a menos que se establezcan explícitamente `security=full` y `ask=off`.

## Aprobaciones de ejecución (aplicación complementaria / host del nodo)

Los agentes aislados pueden requerir aprobación para cada solicitud antes de que `exec` se ejecute en el Gateway o en el host del nodo. Consulte [Aprobaciones de ejecución](/es/tools/exec-approvals) para conocer la política, la lista de permitidos y el flujo de la interfaz de usuario.

Cuando se requiere aprobación humana, los flujos del host del nodo y los flujos no nativos del Gateway devuelven inmediatamente `status: "approval-pending"` y un identificador de aprobación. En cambio, los flujos del chat nativo y de la interfaz web del Gateway pueden esperar en línea y devolver el resultado final del comando tras su aprobación. Un resultado `approval-pending` significa que el comando no se ha iniciado, por lo que las advertencias de reserva de primer plano solo aparecen si el comando aprobado se ejecuta realmente en línea. Las ejecuciones asíncronas aprobadas emiten eventos del sistema de progreso y finalización del comando (`Exec running` / `Exec finished`); las aprobaciones denegadas o agotadas son terminales y no reactivan la sesión del agente con un evento del sistema de denegación.

En los canales con tarjetas o botones de aprobación nativos, el agente debe usar primero esa interfaz nativa e incluir un comando manual `/approve` únicamente cuando el resultado de la herramienta indique explícitamente que las aprobaciones mediante chat no están disponibles o que la aprobación manual es la única vía.

## Lista de permitidos y binarios seguros

La aplicación manual de la lista de permitidos compara patrones glob de rutas binarias resueltas y patrones glob de nombres de comandos sin ruta. Los nombres sin ruta solo coinciden con comandos invocados mediante PATH, por lo que `rg` puede coincidir con `/opt/homebrew/bin/rg` cuando el comando es `rg`, pero no con `./rg` ni `/tmp/rg`.

Cuando `security=allowlist`, los comandos del shell solo se permiten automáticamente si cada segmento de la canalización está en la lista de permitidos o es un binario seguro. El encadenamiento (`;`, `&&`, `||`) y las redirecciones se rechazan en el modo de lista de permitidos a menos que cada segmento de nivel superior cumpla la lista de permitidos (incluidos los binarios seguros). Las redirecciones siguen sin ser compatibles. La confianza permanente `allow-always` no elude esta regla: un comando encadenado sigue requiriendo que cada segmento de nivel superior coincida.

`autoAllowSkills` es una vía práctica independiente de las aprobaciones de ejecución y no equivale a las entradas manuales de rutas de la lista de permitidos. Para una confianza explícita y estricta, mantenga `autoAllowSkills` desactivado.

Use los dos controles para fines distintos:

- `tools.exec.safeBins`: filtros pequeños de flujos que solo reciben datos por stdin.
- `tools.exec.safeBinTrustedDirs`: directorios adicionales de confianza explícita para las rutas ejecutables de binarios seguros.
- `tools.exec.safeBinProfiles`: política explícita de argv para binarios seguros personalizados.
- lista de permitidos: confianza explícita en rutas ejecutables.

No trate `safeBins` como una lista de permitidos genérica ni añada binarios de intérpretes o entornos de ejecución (por ejemplo, `python3`, `node`, `ruby`, `bash`). Si los necesita, use entradas explícitas de la lista de permitidos y mantenga activadas las solicitudes de aprobación.

`openclaw security audit` advierte cuando las entradas `safeBins` de intérpretes o entornos de ejecución carecen de perfiles explícitos, y `openclaw doctor --fix` puede generar la estructura inicial de las entradas personalizadas `safeBinProfiles` que falten. `openclaw security audit` y `openclaw doctor` también advierten cuando se vuelven a añadir explícitamente binarios con un comportamiento amplio, como `jq`, a `safeBins` (`jq` puede leer datos del entorno y cargar código jq desde módulos o archivos de inicio, por lo que se recomienda utilizar entradas explícitas de la lista de permitidos o ejecuciones sujetas a aprobación). `jq` se deniega como binario seguro incluso cuando aparece explícitamente en la lista. Si se incluyen explícitamente intérpretes en la lista de permitidos, active `tools.exec.strictInlineEval` para que las formas de evaluación de código en línea sigan requiriendo revisión o aprobación explícita.

Para obtener todos los detalles y ejemplos de la política, consulte [Aprobaciones de ejecución](/es/tools/exec-approvals-advanced#safe-bins-stdin-only) y [Binarios seguros frente a lista de permitidos](/es/tools/exec-approvals-advanced#safe-bins-versus-allowlist).

## Ejemplos

Primer plano:

```json
{ "tool": "exec", "command": "ls -la" }
```

Segundo plano y consulta:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

La consulta sirve para obtener el estado bajo demanda, no para crear bucles de espera. Si está activada la reactivación automática al finalizar, el comando puede reactivar la sesión cuando emita una salida o falle.

Enviar teclas (al estilo de tmux):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Enviar (solo envía CR):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Pegar (con delimitación de forma predeterminada):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` es una subherramienta de `exec` para realizar ediciones estructuradas en varios archivos. Está activada de forma predeterminada y disponible para cualquier proveedor de modelos; `allowModels` puede restringirla. Use la configuración únicamente cuando desee desactivarla o limitarla a modelos específicos:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

Notas:

- La política de herramientas sigue aplicándose; `allow: ["write"]` permite implícitamente `apply_patch`.
- `deny: ["write"]` no deniega `apply_patch`; deniegue `apply_patch` explícitamente o use `deny: ["group:fs"]` cuando también deban bloquearse las escrituras de parches.
- La configuración se encuentra en `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` tiene como valor predeterminado `true`; establézcalo en `false` para desactivar la herramienta.
- `tools.exec.applyPatch.workspaceOnly` tiene como valor predeterminado `true` (limitado al espacio de trabajo). Establézcalo en `false` únicamente si se desea intencionadamente que `apply_patch` escriba o elimine contenido fuera del directorio del espacio de trabajo.
- `tools.exec.applyPatch.allowModels` es una lista de permitidos opcional de identificadores de modelos (sin procesar, como `gpt-5.4`, o completos, como `openai/gpt-5.4`). Cuando se establece, solo los modelos coincidentes reciben la herramienta; cuando no se establece, todos los modelos la reciben.

## Temas relacionados

- [Aprobaciones de ejecución](/es/tools/exec-approvals) — puertas de aprobación para comandos del shell
- [Aislamiento](/es/gateway/sandboxing) — ejecución de comandos en entornos aislados
- [Proceso en segundo plano](/es/gateway/background-process) — herramientas de ejecución y procesos de larga duración
- [Seguridad](/es/gateway/security) — política de herramientas y acceso elevado
