---
read_when:
    - Configuración de aprobaciones o listas de permitidos para exec
    - Implementación de la experiencia de usuario de aprobación de ejecución en la aplicación para macOS
    - Revisión de prompts de escape del sandbox y sus implicaciones
sidebarTitle: Exec approvals
summary: 'Aprobaciones de ejecución en el host: opciones de política, listas de permitidos y el flujo de trabajo YOLO/estricto'
title: Aprobaciones de ejecución
x-i18n:
    generated_at: "2026-07-26T04:53:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Las aprobaciones de ejecución son el **mecanismo de protección de la aplicación complementaria / host Node** que permite que un
agente aislado ejecute comandos en un host real (`gateway` o `node`). Los comandos
solo se ejecutan cuando la política, la lista de permitidos y la aprobación opcional del usuario coinciden.
Las aprobaciones se aplican **además de** la política de herramientas y del control de acceso elevado (el modo elevado
`full` las omite).

Para obtener una descripción general centrada en los modos de `deny`, `allowlist`, `ask`, `auto`, `full`,
la asignación de Codex Guardian y los permisos del entorno de ACPX, consulte
[Modos de permisos](/es/tools/permission-modes).

<Note>
La política efectiva es la **más estricta** entre `tools.exec.*` y los valores
predeterminados de las aprobaciones: las aprobaciones solo pueden reforzar la seguridad o las solicitudes derivadas de la configuración, nunca
relajarlas. Si se omite un campo de aprobaciones, se utiliza el valor `tools.exec`.
La ejecución en el host también utiliza el estado local de aprobaciones de esa máquina: un
`ask: "always"` local del host en el archivo de aprobaciones del host de ejecución mantiene
las solicitudes de confirmación incluso si los valores predeterminados de la sesión o la configuración solicitan `ask: "on-miss"`.
</Note>

## Dónde se aplica

Las aprobaciones de ejecución se aplican localmente en el host de ejecución:

- **Host del Gateway** -> proceso `openclaw` en la máquina del Gateway.
- **Host Node** -> ejecutor del nodo (aplicación complementaria de macOS o host Node sin interfaz gráfica).

### Modelo de confianza

- Quienes realizan llamadas autenticadas por el Gateway son operadores de confianza de ese Gateway.
- Los nodos emparejados extienden esa capacidad del operador de confianza al host Node.
- Las aprobaciones reducen el riesgo de ejecución accidental, pero **no** constituyen un límite de autenticación por usuario ni una política de solo lectura del sistema de archivos.
- Una vez aprobado, un comando puede modificar archivos de acuerdo con los permisos del sistema de archivos del host o entorno aislado seleccionado.
- Las ejecuciones aprobadas en el host Node vinculan el contexto de ejecución canónico: directorio de trabajo, argumentos exactos, vinculación del entorno cuando exista y ruta fijada del ejecutable cuando corresponda.
- Para scripts de shell e invocaciones directas de archivos mediante intérpretes o entornos de ejecución, OpenClaw también intenta vincular un operando concreto de archivo local. Si ese archivo cambia después de la aprobación, pero antes de la ejecución, se deniega la ejecución en lugar de ejecutar contenido modificado.
- La vinculación de archivos es una medida de mejor esfuerzo, no un modelo completo de todas las rutas de carga de cada intérprete o entorno de ejecución. Si no se puede identificar exactamente un archivo local concreto, OpenClaw se niega a generar una ejecución respaldada por una aprobación en lugar de simular una cobertura total.

### Separación en macOS

- El **servicio del host Node** reenvía `system.run` a la **aplicación de macOS** mediante IPC local.
- La **aplicación de macOS** aplica las aprobaciones y ejecuta el comando en el contexto de la interfaz de usuario.

## Inspección de la política efectiva

| Comando                                                          | Qué muestra                                                                             |
| ---------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | La política solicitada, las fuentes de la política del host y el resultado efectivo.    |
| `openclaw exec-policy show`                                      | La vista combinada de la máquina local.                                                  |
| `openclaw exec-policy set` / `preset`                            | Sincroniza en un solo paso la política local solicitada con el archivo local de aprobaciones del host. |

<Note>
Las anulaciones `/exec` por sesión no se incluyen. Ejecute `/exec` en la sesión correspondiente para inspeccionar sus valores predeterminados actuales. Consulte [anulaciones de sesión](/es/tools/exec#session-overrides-exec).
</Note>

Referencia completa de la CLI (opciones, salida JSON y adición o eliminación de elementos de la lista de permitidos): [CLI de aprobaciones](/es/cli/approvals).

Cuando un ámbito local solicita `host=node`, `exec-policy show` indica que
ese ámbito está administrado por el nodo durante la ejecución, en lugar de tratar el archivo local de aprobaciones
como la fuente de verdad.

Si la interfaz de usuario de la aplicación complementaria **no está disponible**, cualquier solicitud que
normalmente requiriese confirmación se resuelve mediante el **mecanismo alternativo de solicitud** (valor predeterminado: `deny`).

<Tip>
Los clientes nativos de aprobación por chat pueden proporcionar acciones específicas del canal en el
mensaje de aprobación pendiente. Matrix proporciona atajos mediante reacciones (`✅` permitir una vez,
`♾️` permitir siempre, `❌` denegar), sin dejar de incluir `/approve ...` en el
mensaje como alternativa.
</Tip>

## Configuración y almacenamiento

Las aprobaciones se guardan en un archivo JSON local del host de ejecución. Cuando
se establece `OPENCLAW_STATE_DIR`, el archivo sigue ese directorio de estado;
de lo contrario, utiliza el directorio de estado predeterminado de OpenClaw:

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# de lo contrario
~/.openclaw/exec-approvals.json
```

El socket de aprobación predeterminado utiliza la misma raíz:
`$OPENCLAW_STATE_DIR/exec-approvals.sock`, o
`~/.openclaw/exec-approvals.sock` cuando la variable no está definida.

Los directorios de estado son ámbitos de confianza independientes. Cuando `OPENCLAW_STATE_DIR`
apunta a otra ubicación, OpenClaw nunca importa ni archiva
`~/.openclaw/exec-approvals.json`; configure las aprobaciones por separado para el
directorio de estado personalizado. Doctor también importa el elemento heredado
`plugin-binding-approvals.json` únicamente cuando pertenece al directorio de estado
activo.

Ejemplo de esquema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## Controles de políticas

### `tools.exec.mode`

`tools.exec.mode` es la superficie de políticas normalizada preferida para la ejecución en el host:

| Valor       | Comportamiento                                                                                                                                                                                              |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | Bloquea la ejecución en el host.                                                                                                                                                                             |
| `allowlist` | Ejecuta únicamente comandos incluidos en la lista de permitidos sin solicitar confirmación.                                                                                                                 |
| `ask`       | Utiliza la política de lista de permitidos y solicita confirmación cuando no hay coincidencias.                                                                                                              |
| `auto`      | Utiliza la política de lista de permitidos, ejecuta directamente las coincidencias deterministas y envía los casos sin aprobación al revisor automático nativo de OpenClaw antes de recurrir a una aprobación humana. |
| `full`      | Ejecuta comandos en el host sin solicitudes de aprobación.                                                                                                                                                   |

Doctor migra el par persistente retirado `tools.exec.security` / `tools.exec.ask`
a `tools.exec.mode`.

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - bloquea todas las solicitudes de ejecución en el host.
  - `allowlist` - permite únicamente los comandos incluidos en la lista de permitidos.
  - `full` - permite todo (equivalente al modo elevado).

El valor predeterminado es `full` para los hosts del Gateway o Node; en cambio, un host `sandbox` utiliza
`deny` de forma predeterminada.
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  Política de solicitud configurada para la ejecución en el host. Controla el comportamiento básico de las
  solicitudes de aprobación procedentes de `tools.exec.ask` y de los valores predeterminados de las aprobaciones del host.
  El valor predeterminado es `off`. El parámetro `ask` por llamada de la herramienta (consulte
  [Herramienta de ejecución](/es/tools/exec#parameters)) solo puede reforzar ese valor de referencia, y
  las llamadas del modelo originadas en un canal lo ignoran cuando la solicitud efectiva del host es `off`.

- `off` - nunca solicita confirmación.
- `on-miss` - solicita confirmación únicamente cuando la lista de permitidos no coincide.
- `always` - solicita confirmación para cada comando. La confianza persistente de `allow-always` **no** suprime las solicitudes cuando el modo efectivo de solicitud es `always`.

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  Resolución cuando se requiere una solicitud, pero no hay ninguna interfaz de usuario accesible (o se
  agota el tiempo de espera de la solicitud). Si se omite, el valor predeterminado es `deny`.

- `deny` - bloquea.
- `allowlist` - permite únicamente si la lista de permitidos coincide.
- `full` - permite.

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  Cuando se establece en `true`, trata las formas de evaluación de código en línea como sujetas exclusivamente a aprobación, incluso si el
  propio binario del intérprete está incluido en la lista de permitidos. Proporciona defensa en profundidad para los
  cargadores de intérpretes que no se corresponden claramente con un único operando de archivo estable.
</ParamField>

Ejemplos detectados por el modo estricto: `python -c`, `node -e`/`--eval`/`-p`,
`ruby -e`, `perl -e`/`-E`, `php -r`, `lua -e`, `osascript -e` (también las formas en línea
`awk`, `sed`, `make`, `find -exec` y `xargs`).

En modo estricto, estos comandos requieren la revisión o la aprobación explícita. Con
`tools.exec.mode: "auto"`, el revisor puede autorizar una ejecución de bajo riesgo cuando
el comando tiene un plan aplicable; de lo contrario, OpenClaw solicita la aprobación de una persona.
Las aprobaciones de comandos `Codex app-server` que llegan al mecanismo alternativo del revisor solicitan la intervención de una
persona porque sus solicitudes de aprobación no presentan un ejecutable resuelto que se pueda aplicar.
`allow-always` no conserva nuevas entradas de la lista de permitidos para comandos de evaluación en línea.

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  Solo afecta a la presentación: cuando está habilitado, OpenClaw puede adjuntar
  segmentos de comandos derivados del analizador para que las solicitudes de aprobación web puedan resaltar los tokens de los comandos. **No**
  cambia `security`, `ask`, la correspondencia con la lista de permitidos, el comportamiento estricto de la evaluación en línea,
  el reenvío de aprobaciones ni la ejecución de comandos.
</ParamField>

Establézcalo globalmente en `tools.exec.commandHighlighting` o por agente en
`agents.entries.*.tools.exec.commandHighlighting`.

## Modo YOLO (sin aprobación)

Para ejecutar comandos en el host sin solicitudes de aprobación, abra **ambas** capas de políticas:
la política de ejecución solicitada en la configuración de OpenClaw (`tools.exec.*`) **y**
la política de aprobaciones local del host en el archivo de aprobaciones del host de ejecución.

Si se omite `askFallback`, el valor predeterminado es `deny`. Establezca explícitamente `askFallback` del host en `full`
cuando una solicitud de aprobación sin interfaz de usuario deba recurrir a permitir.

| Capa               | Configuración YOLO                        |
| ------------------ | ----------------------------------------- |
| `tools.exec.mode`  | `full` en `gateway`/`node` |
| `askFallback` del host | `full`                          |

<Warning>
**Distinciones importantes:**

- `tools.exec.host=auto` elige **dónde** se ejecuta exec: en el sandbox cuando está disponible; de lo contrario, en el Gateway.
- YOLO elige **cómo** se aprueba la ejecución en el host: `security=full` más `ask=off`.
- YOLO **no** añade una puerta de aprobación heurística independiente para la ofuscación de comandos ni una capa de rechazo de comprobación previa de scripts además de la política de ejecución en el host configurada.
- `auto` no convierte el enrutamiento al Gateway en una opción que pueda sustituirse libremente desde una sesión en sandbox. Se permite una solicitud `host=node` por llamada desde `auto`; `host=gateway` solo se permite desde `auto` cuando no hay ningún entorno de ejecución de sandbox activo. Para establecer un valor predeterminado estable y no automático, configure `tools.exec.host` o use `/exec host=...` explícitamente.

</Warning>

Los proveedores respaldados por la CLI que exponen su propio modo de permisos no interactivo
pueden seguir esta política. La CLI de Claude añade
`--permission-mode bypassPermissions` cuando la política de ejecución efectiva
de OpenClaw es YOLO. En las sesiones en vivo de Claude gestionadas por OpenClaw, la
política de ejecución efectiva de OpenClaw prevalece sobre el modo de permisos nativo de Claude:
YOLO normaliza los inicios de sesiones en vivo a `--permission-mode bypassPermissions`, y
una política de ejecución efectiva restrictiva normaliza los inicios de sesiones en vivo a
`--permission-mode default`, incluso si los argumentos sin procesar del backend de Claude especifican otro
modo.

Si se desea una configuración más conservadora, restrinja de nuevo la política de ejecución de OpenClaw a
`allowlist` / `on-miss` o `deny`.

### Configuración persistente de «no solicitar nunca» para el host del Gateway

<Steps>
  <Step title="Establecer la política de configuración solicitada">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="Hacer coincidir el archivo de aprobaciones del host">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### Atajo local

```bash
openclaw exec-policy preset yolo
```

Actualiza tanto `tools.exec.host/security/ask` local como los valores predeterminados del archivo de
aprobaciones local (incluido `askFallback: "full"`). Es intencionadamente
solo local. Para cambiar de forma remota las aprobaciones del host del Gateway o del host del Node, use
`openclaw approvals set --gateway` o `openclaw approvals set --node
<id|name|ip>`.

Otros ajustes preestablecidos integrados: `cautious` (`host=gateway`, `security=allowlist`,
`ask=on-miss`, `askFallback=deny`) y `deny-all` (`host=gateway`,
`security=deny`, `ask=off`, `askFallback=deny`). Aplíquelos de la misma manera:
`openclaw exec-policy preset cautious`.

Para establecer campos individuales en lugar de un ajuste preestablecido completo, use
`openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>` con cualquier subconjunto de esas opciones.

### Host del Node

Aplique en su lugar el mismo archivo de aprobaciones en el Node:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**Limitaciones de solo local:**

- `openclaw exec-policy` no sincroniza las aprobaciones del Node.
- `openclaw exec-policy set --host node` se rechaza.
- Las aprobaciones de ejecución del Node se obtienen del Node durante la ejecución, por lo que las actualizaciones dirigidas al Node deben usar `openclaw approvals --node ...`.

</Note>

### Atajo solo para la sesión

- `/exec security=full ask=off` cambia únicamente la sesión actual.
- `/elevated full` es un atajo de emergencia que omite las aprobaciones de ejecución solo
  cuando tanto la política solicitada como el archivo de aprobaciones del host se resuelven como
  `security: "full"` y `ask: "off"`. Un archivo del host más estricto, como `ask:
"always"`, continúa solicitando confirmación.

Si el archivo de aprobaciones del host sigue siendo más estricto que la configuración, la política
más estricta del host continúa prevaleciendo.

## Lista de permitidos (por agente)

Las listas de permitidos son **por agente**. Si hay varios agentes, cambie el agente
que está editando en la aplicación de macOS. Los patrones se comparan mediante glob.

Los patrones pueden ser globs de rutas de binarios resueltas o globs de nombres de comandos sin ruta.
Los nombres sin ruta solo coinciden con comandos invocados mediante `PATH`, por lo que `rg` puede coincidir con
`/opt/homebrew/bin/rg` cuando el comando es `rg`, pero **no** con `./rg` ni
`/tmp/rg`. Use un glob de ruta para confiar en una ubicación específica del binario.

Las entradas `agents.default` heredadas se migran a `agents.main` al cargarse.
Las cadenas de shell como `echo ok && pwd` siguen requiriendo que cada segmento de nivel superior
cumpla las reglas de la lista de permitidos.

Ejemplos:

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### Restricción de argumentos con argPattern

Añada `argPattern` cuando una entrada de la lista de permitidos deba coincidir con un binario y una
forma específica de argumentos. OpenClaw utiliza la semántica de expresiones regulares de ECMAScript
(JavaScript) en todos los hosts y evalúa la expresión con respecto a los argumentos analizados del comando,
sin incluir el token del ejecutable (`argv[0]`).
En las entradas creadas manualmente, los argumentos se unen con un solo espacio; por tanto,
ancle el patrón cuando necesite una coincidencia exacta.

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

Esa entrada permite `python3 safe.py`; `python3 other.py` no coincide con la lista de
permitidos. Si también existe una entrada de solo ruta para el mismo binario, los argumentos que no coincidan
aún pueden recurrir a esa entrada de solo ruta. Omita la entrada de solo ruta
cuando el objetivo sea restringir el binario a los argumentos declarados.

Las entradas guardadas por los flujos de aprobación utilizan un formato interno con separadores para la coincidencia
exacta de argv. Es preferible usar la interfaz o el flujo de aprobación para regenerar esas entradas
en lugar de editar manualmente el valor codificado. Si OpenClaw no puede analizar argv
para un segmento de comando, las entradas con `argPattern` no coinciden.

Las entradas `allow-always` generadas están vinculadas a argv. Las nuevas entradas generadas incluyen
`argPattern`; las entradas antiguas generadas de solo ruta se ignoran y necesitan una nueva
aprobación. Para una regla manual de solo ruta, omita tanto `source` como `argPattern`.

Cada entrada de la lista de permitidos admite:

| Campo              | Significado                                                                  |
| ------------------ | ------------------------------------------------------------------------ |
| `pattern`          | Glob de ruta de binario resuelta o glob de nombre de comando sin ruta                      |
| `argPattern`       | Expresión regular de argv de ECMAScript o hash exacto de argv generado; si se omite, es solo de ruta |
| `id`               | ID opaco estable; se genera como UUID cuando no está presente                        |
| `source`           | Origen de la entrada generada, como `allow-always`; se omite en las entradas manuales  |
| `commandText`      | Entrada heredada en texto sin formato; se descarta durante la carga                            |
| `lastUsedAt`       | Marca de tiempo del último uso                                                      |
| `lastUsedCommand`  | Último comando que coincidió; se omite en las entradas de argv con hash generadas     |
| `lastResolvedPath` | Última ruta de binario resuelta                                                |

## Permitir automáticamente las CLI de Skills

Cuando **Permitir automáticamente las CLI de Skills** (`autoAllowSkills`) está activado, los ejecutables
a los que hacen referencia las Skills conocidas se consideran incluidos en la lista de permitidos de los Nodes (Node de macOS
o host de Node sin interfaz gráfica). Esto utiliza `skills.bins` mediante la RPC del Gateway para
obtener la lista de binarios de las Skills. Desactive esta opción si desea listas de permitidos
manuales estrictas.

<Warning>
- Esta es una **lista de permitidos implícita por comodidad**, independiente de las entradas manuales de la lista de permitidos por ruta.
- Está pensada para entornos de operadores de confianza en los que el Gateway y el Node se encuentran dentro del mismo límite de confianza.
- Si se requiere una confianza explícita estricta, mantenga `autoAllowSkills: false` y use únicamente entradas manuales de la lista de permitidos por ruta.

</Warning>

## Binarios seguros y reenvío de aprobaciones

Para obtener información sobre los binarios seguros (la vía rápida de solo stdin), los detalles de vinculación
del intérprete y cómo reenviar solicitudes de aprobación a Slack/Discord/Telegram (o ejecutarlas como
clientes de aprobación nativos), consulte
[Aprobaciones de ejecución: opciones avanzadas](/es/tools/exec-approvals-advanced).

## Edición en la interfaz de control

Use la tarjeta **Control UI -> Nodes -> Exec approvals** para editar los valores predeterminados,
las sustituciones por agente y las listas de permitidos. Elija un ámbito (Defaults o un agente),
ajuste la política, añada o elimine patrones de la lista de permitidos y, a continuación, seleccione **Save**. La interfaz
muestra los metadatos del último uso de cada patrón para facilitar el mantenimiento de la lista.

El selector de destino permite elegir **Gateway** (aprobaciones locales) o un **Node**.
Los Nodes deben anunciar `system.execApprovals.get/set` (aplicación de macOS o
host de Node sin interfaz gráfica). Si un Node aún no anuncia las aprobaciones de ejecución, edite directamente su
archivo de aprobaciones local.

Algunos hosts de Node, incluido el complemento de Windows, utilizan un formato de política de aprobación
diferente. La interfaz de control muestra estas políticas nativas del host en modo de solo lectura. Use la
aplicación complementaria o `openclaw approvals set --node <id|name|ip>` con la forma de
política nativa para editarlas; consulte [CLI de aprobaciones](/es/cli/approvals).

CLI: `openclaw approvals` permite editar el Gateway o el Node; consulte
[CLI de aprobaciones](/es/cli/approvals).

## Flujo de aprobación

Cuando se requiere una solicitud de confirmación, el Gateway transmite
`exec.approval.requested` a los clientes operadores. La interfaz de control y la aplicación de macOS
la resuelven mediante `exec.approval.resolve`; a continuación, el Gateway reenvía la
solicitud aprobada al host del Node.

Para `host=node`, las solicitudes de aprobación incluyen una carga útil canónica `systemRunPlan`.
El Gateway usa ese plan como contexto autoritativo del comando, cwd y la sesión
al reenviar las solicitudes `system.run` aprobadas:

- La ruta de ejecución del Node prepara inicialmente un único plan canónico.
- El registro de aprobación almacena ese plan y sus metadatos de vinculación.
- Una vez aprobado, la llamada `system.run` final reenviada reutiliza el plan almacenado en lugar de confiar en ediciones posteriores del autor de la llamada.
- Si el autor de la llamada cambia `command`, `rawCommand`, `cwd`, `agentId` o `sessionKey` después de crear la solicitud de aprobación, el Gateway rechaza la ejecución reenviada porque la aprobación no coincide.

## Eventos del sistema y denegaciones

El ciclo de vida de exec publica un mensaje del sistema `Exec finished` en la sesión
del agente después de que el Node informa de la finalización. OpenClaw también puede emitir un
aviso de ejecución en curso una vez concedida una aprobación, después de que
transcurra `tools.exec.approvalRunningNoticeMs` (valor predeterminado: `10000`; `0` lo desactiva).
Las aprobaciones de ejecución denegadas son terminales para el comando del host: el comando
no se ejecuta.

- En las aprobaciones asíncronas del agente principal que tienen una sesión de origen, OpenClaw
  publica la denegación en esa sesión como seguimiento interno para que el
  agente pueda dejar de esperar el comando asíncrono y evitar una reparación
  por resultado ausente.
- Si no hay ninguna sesión o no se puede reanudar, OpenClaw aún puede
  comunicar una denegación concisa al operador o a la ruta de chat directo.
- Las denegaciones de las sesiones de subagentes y Cron no se publican en esas
  sesiones.

Las aprobaciones de ejecución del host del Gateway emiten el mismo evento de finalización del ciclo de vida.
Las ejecuciones sujetas a aprobación reutilizan el ID de aprobación para correlacionar la solicitud
pendiente con su mensaje de finalización o denegación (`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`).

## Implicaciones

- **`full`** es potente; es preferible usar listas de permitidos cuando sea posible.
- **`ask`** permite mantener el control y, al mismo tiempo, aprobar con rapidez.
- Las listas de permitidos por agente impiden que las aprobaciones de un agente se filtren a otros.
- Las aprobaciones solo se aplican a solicitudes de ejecución en el host de **remitentes autorizados**. Los remitentes no autorizados no pueden emitir `/exec`.
- `/exec security=full` es una comodidad a nivel de sesión para operadores autorizados y omite las aprobaciones intencionadamente. Para bloquear por completo la ejecución en el host, establezca la seguridad de las aprobaciones en `deny` o deniegue la herramienta `exec` mediante la política de herramientas.

## Temas relacionados

<CardGroup cols={2}>
  <Card title="Aprobaciones de ejecución: avanzadas" href="/es/tools/exec-approvals-advanced" icon="gear">
    Binarios seguros, vinculación de intérpretes y reenvío de aprobaciones al chat.
  </Card>
  <Card title="Herramienta de ejecución" href="/es/tools/exec" icon="terminal">
    Herramienta de ejecución de comandos de shell.
  </Card>
  <Card title="Modo elevado" href="/es/tools/elevated" icon="shield-exclamation">
    Vía de emergencia que también omite las aprobaciones.
  </Card>
  <Card title="Aislamiento" href="/es/gateway/sandboxing" icon="box">
    Modos de aislamiento y acceso al espacio de trabajo.
  </Card>
  <Card title="Seguridad" href="/es/gateway/security" icon="lock">
    Modelo de seguridad y refuerzo.
  </Card>
  <Card title="Aislamiento frente a política de herramientas frente a modo elevado" href="/es/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    Cuándo recurrir a cada control.
  </Card>
  <Card title="Skills" href="/es/tools/skills" icon="sparkles">
    Comportamiento de autorización automática basado en Skills.
  </Card>
</CardGroup>
