---
read_when:
    - Quieres inspeccionar o crear tarjetas de Workboard desde la terminal
    - Quieres iniciar ejecuciones de trabajadores de Workboard desde la CLI
    - Estás depurando el comportamiento de la CLI o de los comandos con barra de Workboard
summary: Referencia de la CLI para tarjetas de `openclaw workboard`, despacho y ejecuciones de trabajadores
title: CLI de Workboard
x-i18n:
    generated_at: "2026-07-26T04:37:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 640260ea6f5959b3aee1cdce76f2501097bff79e9bf1741bdd9ff7a8b43e1a7f
    source_path: cli/workboard.md
    workflow: 16
---

`openclaw workboard` es la interfaz de terminal del [plugin Workboard](/es/plugins/workboard) incluido. Permite a un operador enumerar tarjetas, crear una tarjeta, inspeccionar una tarjeta y solicitar al Gateway en ejecución que despache el trabajo listo a ejecuciones de trabajadores subagente.

Habilite el plugin antes de usar el comando:

```bash
openclaw plugins enable workboard
openclaw gateway restart
```

## Uso

```bash
openclaw workboard list [--board <id>] [--status <status>] [--include-archived] [--json]
openclaw workboard create <title...> [--notes <text>] [--status <status>] [--priority <priority>] [--agent <id>] [--board <id>] [--labels <items>] [--json]
openclaw workboard show <id> [--json]
openclaw workboard move <id> --status <status> [--json]
openclaw workboard dispatch [--board <id>] [--max-starts <count>] [--admin] [--url <url>] [--token <token>] [--timeout <ms>] [--json]
```

El comando lee y escribe en la misma base de datos SQLite propiedad del plugin que utilizan el panel y las herramientas del agente de Workboard. Los identificadores de las tarjetas son UUID; los comandos que aceptan el identificador de una tarjeta también aceptan un prefijo de identificador inequívoco (la salida de texto compacta muestra los primeros 8 caracteres).

Valores válidos de `status`: `triage`, `backlog`, `todo`, `scheduled`, `ready`, `running`, `review`, `blocked`, `done`. Valores válidos de `priority`: `low`, `normal`, `high`, `urgent`.

## `list`

```bash
openclaw workboard list
openclaw workboard list --board default --status ready
openclaw workboard list --json
```

La salida de texto es compacta:

```text
7f4a2c10  ready     high    default agent-a  Corregir el Heartbeat obsoleto del trabajador
```

Las columnas son el prefijo del identificador, el estado, la prioridad, el identificador del tablero, el identificador opcional del agente y el título.

| Opción                 | Finalidad                                       |
| -------------------- | --------------------------------------------- |
| `--board <id>`       | Limitar los resultados al espacio de nombres de un tablero          |
| `--status <status>`  | Limitar los resultados a un estado de Workboard         |
| `--include-archived` | Incluir tarjetas archivadas en la salida de texto compacta |
| `--json`             | Imprimir la lista completa de tarjetas como JSON para máquinas      |

La salida de texto compacta oculta las tarjetas archivadas de forma predeterminada para que la CLI coincida con `/workboard list`. Pase `--include-archived` para mostrarlas. La salida JSON siempre conserva la lista completa de tarjetas, incluidas las archivadas, para la automatización existente.

## `create`

```bash
openclaw workboard create "Fix stale worker heartbeat" --priority high --labels bug,workboard
openclaw workboard create "Write Workboard docs" --status ready --agent docs-agent --board docs --notes "Cover CLI, slash command, dispatch, and SQLite state."
```

| Opción                    | Finalidad                                 |
| ----------------------- | --------------------------------------- |
| `--notes <text>`        | Notas iniciales de la tarjeta                      |
| `--status <status>`     | Estado inicial, valor predeterminado `todo`          |
| `--priority <priority>` | Prioridad, valor predeterminado `normal`              |
| `--agent <id>`          | Asignar la tarjeta a un agente o identificador de propietario |
| `--board <id>`          | Almacenar la tarjeta en el espacio de nombres de un tablero     |
| `--labels <items>`      | Etiquetas separadas por comas                  |
| `--json`                | Imprimir la tarjeta creada como JSON para máquinas  |

`create` escribe directamente en el estado SQLite de Workboard. La tarjeta aparece de inmediato en la pestaña Workboard de la interfaz de control y para las herramientas de Workboard.

## `show`

```bash
openclaw workboard show 7f4a2c10
openclaw workboard show 7f4a2c10 --json
```

La salida de texto imprime la línea compacta de la tarjeta y las notas. La salida JSON devuelve el registro completo de la tarjeta, incluidos los metadatos de ejecución, intentos, comentarios, enlaces, pruebas, artefactos, registros del trabajador, estado del protocolo, diagnósticos y metadatos de automatización.

Los estados de las pruebas en JSON son resultados informados por el trabajador. `passed` registra la
autoevaluación del trabajador sobre el comando o la comprobación adjuntos; no es un resultado
de verificación independiente.

## `move`

```bash
openclaw workboard move 7f4a2c10 --status review
openclaw workboard move 7f4a2c10 --status done --json
```

`move` cambia el estado de la tarjeta mediante la misma ruta de operador manual que se utiliza al arrastrar una tarjeta en el panel. Acepta el identificador completo de una tarjeta o un prefijo inequívoco. Las retenciones activas por dependencias y programación siguen aplicándose. Los operadores pueden mover una tarjeta reclamada sin el token de reclamación de su agente; los tokens de reclamación siguen limitados a las mutaciones mediante herramientas del agente y se ocultan en la salida JSON.

## `dispatch`

```bash
openclaw workboard dispatch
openclaw workboard dispatch --json
openclaw workboard dispatch --max-starts 10
openclaw workboard dispatch --admin
openclaw workboard dispatch --url http://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

`dispatch` primero llama al método RPC `workboard.cards.dispatch` del Gateway en ejecución, que utiliza el mismo entorno de ejecución de subagentes que la acción de despacho del panel, por lo que las tarjetas listas se convierten en ejecuciones de trabajadores con seguimiento de tareas y claves de sesión vinculadas. `--max-starts` utiliza el método aditivo `workboard.cards.dispatchWithOptions` para que un Gateway anterior rechace la opción antes de iniciar trabajadores; reinicie el Gateway después de actualizar antes de usar la opción. Las tarjetas con un agente asignado utilizan claves de sesión de subagente limitadas al agente; las tarjetas sin asignar conservan una clave de subagente sin ámbito para mantener el agente predeterminado configurado del Gateway.

El bucle de despacho:

1. Promueve a `ready` los elementos secundarios cuyas dependencias están listas.
2. Bloquea las reclamaciones vencidas o las ejecuciones de trabajadores que han agotado el tiempo de espera.
3. Registra metadatos de despacho en las tarjetas listas.
4. Selecciona un pequeño lote de tarjetas listas sin reclamar.
5. Reclama cada tarjeta seleccionada para el despachador o el agente asignado.
6. Inicia una ejecución de trabajador subagente con un contexto de tarjeta limitado y el token de reclamación de la tarjeta.
7. Almacena en la tarjeta el identificador de la ejecución del trabajador, la clave de sesión, la vinculación de tareas cuando el registro de tareas del Gateway la informa, el estado de ejecución y el registro del trabajador.

La selección es conservadora: cada despacho inicia como máximo tres trabajadores de forma predeterminada, omite las tarjetas archivadas o ya reclamadas e inicia solo una tarjeta por propietario o agente en una única pasada. Las tarjetas que ya pertenezcan a trabajos activos en ejecución o en revisión se dejan para un despacho posterior. Pase `--max-starts <count>` con un entero positivo para cambiar el límite por pasada; la regla de una tarjeta por propietario sigue aplicándose, por lo que el número efectivo de inicios puede ser menor.

Si el inicio del trabajador falla después de reclamar una tarjeta, Workboard bloquea esa tarjeta, elimina la reclamación y registra el fallo en los metadatos de ejecución y del registro del trabajador de la tarjeta, lo que mantiene visibles los inicios fallidos en lugar de devolver silenciosamente la tarjeta a la cola.

Si no se proporciona un destino explícito del Gateway y el Gateway local no está disponible o todavía no expone el método de despacho de Workboard, la CLI recurre al despacho exclusivo de datos sobre el estado local de Workboard. El despacho exclusivo de datos aún puede promover dependencias, limpiar reclamaciones obsoletas y bloquear ejecuciones que hayan agotado el tiempo de espera, pero no inicia trabajadores. Los fallos de autenticación, permisos y validación, así como los fallos de un destino `--url` o `--token` explícito, se notifican directamente en lugar de activar la alternativa.

La salida de texto informa de los inicios de trabajadores:

```text
despacho completado: iniciados=2 fallos=0
```

La salida alternativa es explícita:

```text
gateway no disponible; solo despacho de datos: promovidos=1 bloqueados=0
```

La salida JSON incluye el resultado del despacho. El despacho respaldado por el Gateway puede incluir `started` y `startFailures`; la alternativa exclusiva de datos incluye `gatewayUnavailable: true`. Los tokens de reclamación se ocultan en la salida JSON de las tarjetas.

En el panel, el mismo resultado del despacho se muestra como un resumen breve para que un operador pueda ver cuántas tarjetas se iniciaron, promovieron, bloquearon, volvieron a reclamar o fallaron sin abrir los detalles de las tarjetas.

## Equivalencia con el comando de barra diagonal

Los canales con capacidad para ejecutar comandos pueden usar el comando de barra diagonal equivalente:

```text
/workboard list
/workboard show 7f4a2c10
/workboard create Corregir el Heartbeat obsoleto del trabajador
/workboard move 7f4a2c10 --status review
/workboard dispatch
```

El despacho mediante el comando de barra diagonal también utiliza el entorno de ejecución de subagentes del Gateway, por lo que sigue el mismo comportamiento de reclamación, inicio de trabajadores y fallos que el panel y la ruta del Gateway de la CLI.

`/workboard list` y `/workboard show` son comandos de lectura para remitentes de comandos autorizados. `/workboard create`, `/workboard move` y `/workboard dispatch` modifican el estado del tablero y requieren la condición de propietario en las superficies de chat o un cliente del Gateway con `operator.write` o `operator.admin`.

## Permisos

La ruta de despacho de la CLI normalmente solicita los ámbitos `operator.write` y `operator.read` del Gateway. Las tarjetas vinculadas a un espacio de trabajo se ejecutan directamente en un espacio de trabajo exacto de un agente configurado; una solicitud de árbol de trabajo se limita a ese directorio en lugar de permitir que el host materialice código controlado por el repositorio. El trabajador seleccionado debe tener acceso de escritura a un entorno aislado de Docker no compartido para ese espacio de trabajo exacto, un hash de contenedor activo que coincida con los montajes y la política solicitados y ninguna capacidad para escapar al host. Pase `--admin` para solicitar explícitamente `operator.admin`, permitir otra copia de trabajo del host y utilizar la configuración normal de árboles de trabajo gestionados; la conexión falla si ese ámbito no está aprobado para el cliente. Un token del Gateway de solo lectura puede inspeccionar los datos de Workboard mediante métodos de lectura, pero no puede crear tarjetas ni despachar trabajadores. Los límites del espacio de trabajo no modifican de ningún otro modo el movimiento manual de tarjetas para los invocadores con permiso de modificación de Workboard.

Los comandos locales `list`, `create`, `show` y `move` operan en el directorio de estado local de OpenClaw utilizado por el perfil actual. Use `--dev` o `--profile <name>` en el comando de nivel superior `openclaw` cuando necesite una raíz de estado diferente.

## Solución de problemas

### No aparece ninguna tarjeta

Confirme que el plugin esté habilitado para el mismo perfil y la misma raíz de estado:

```bash
openclaw plugins inspect workboard --runtime --json
```

Si el panel muestra tarjetas pero la CLI no, compruebe que ambos comandos utilicen la misma configuración de `--dev` o `--profile`.

### El despacho indica que es exclusivo de datos

Inicie o reinicie el Gateway:

```bash
openclaw gateway restart
openclaw gateway status --deep
```

Después, vuelva a intentar `openclaw workboard dispatch`. La alternativa exclusiva de datos resulta útil para limpiar el estado local, pero las ejecuciones de trabajadores necesitan un Gateway activo.

### El despacho no inicia nada

Compruebe que haya al menos una tarjeta `ready` sin una reclamación activa:

```bash
openclaw workboard list --status ready
```

Las tarjetas también pueden omitirse cuando el mismo propietario ya tiene trabajo en ejecución o en revisión. Mueva el trabajo completado a `done`, libere las reclamaciones obsoletas mediante las herramientas de Workboard o vuelva a ejecutar el despacho cuando finalice el trabajador activo.

## Contenido relacionado

- [Plugin Workboard](/es/plugins/workboard)
- [Referencia de la CLI](/es/cli)
- [Comandos de barra diagonal](/es/tools/slash-commands)
- [Interfaz de control](/es/web/control-ui)
