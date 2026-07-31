---
read_when:
    - Se necesita una rama y un checkout aislados para una tarea de agente
    - Está configurando tarjetas de Workboard con espacios de trabajo worktree
    - Es necesario restaurar o limpiar un árbol de trabajo gestionado por OpenClaw
summary: Ejecuta tareas de agentes en checkouts de git aislados con instantáneas automáticas y limpieza.
title: Worktrees gestionados
x-i18n:
    generated_at: "2026-07-26T05:08:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98ed2579b7243544dbdb550c4b8a292ccd4ab494fd4a45b2404256691c831401
    source_path: concepts/managed-worktrees.md
    workflow: 16
---

Los árboles de trabajo administrados proporcionan a una tarea de agente su propia rama de git y su propio checkout sin colocar directorios temporales dentro del repositorio de origen. OpenClaw los crea en su directorio de estado, los registra en la base de datos de estado compartida y genera instantáneas de su contenido rastreado y de su contenido no rastreado y no ignorado antes de eliminarlos.

## Disposición y nombres

Cada árbol de trabajo se encuentra en:

```text
<openclaw-state-dir>/worktrees/<repo-fingerprint>/<name>
```

La huella digital del repositorio son los primeros 16 caracteres hexadecimales de un hash SHA-256 calculado sobre el directorio común canónico de git y la URL de origen. El nombre proporcionado debe coincidir con `[a-z0-9][a-z0-9-]{0,63}`. Sin un nombre, OpenClaw genera `wt-` seguido de ocho caracteres hexadecimales aleatorios.

OpenClaw crea la rama `openclaw/<name>` en la referencia base solicitada. Sin una referencia base, obtiene `origin`, utiliza la rama predeterminada remota cuando está disponible y recurre a la rama local `HEAD` cuando el repositorio está sin conexión o no tiene un remoto utilizable.

## Aprovisionamiento de archivos ignorados

Añada `.worktreeinclude` en la raíz del repositorio de origen para copiar determinados archivos ignorados y no rastreados en un árbol de trabajo nuevo. El archivo utiliza la sintaxis de patrones de gitignore, un patrón por línea, con comentarios `#`:

```gitignore
.env.local
fixtures/generated/**
```

Solo son aptos los archivos que git indica como ignorados y no rastreados. Los archivos rastreados ya están presentes mediante git y nunca se copian en este paso. OpenClaw no sobrescribe ni modifica los archivos de destino que ya existen, no sigue directorios enlazados simbólicamente y conserva los modos de los archivos copiados. Registra únicamente las rutas que crea realmente, por lo que las ediciones posteriores del manifiesto no pueden hacer que esos archivos pierdan la protección durante la limpieza.

## Ejecución de la configuración del repositorio

Si `.openclaw/worktree-setup.sh` existe en el repositorio de origen y es ejecutable, OpenClaw lo ejecuta con el árbol de trabajo nuevo como directorio actual. El script recibe:

```text
OPENCLAW_SOURCE_TREE_PATH=<source checkout>
OPENCLAW_WORKTREE_PATH=<managed worktree>
```

Una salida distinta de cero cancela la creación y elimina el árbol de trabajo y la rama nuevos. Este es un contrato local del repositorio; no existe ninguna clave de configuración de OpenClaw para él.

## Árboles de trabajo de sesión

Inicie un chat aislado desde una carpeta respaldada por Git con una sesión de árbol de trabajo: en la página Nueva sesión de la interfaz de control, utilice el selector **Lugar** para elegir una carpeta de origen del Gateway y, a continuación, seleccione **Árbol de trabajo** (con una rama base y un nombre de árbol de trabajo opcionales). La opción solo aparece después de que el Gateway confirme que la carpeta seleccionada es un checkout de Git; las carpetas normales se ejecutan directamente y no muestran ningún control de aislamiento de Git. iOS ofrece la misma opción desde las acciones de Chat, y Android la ofrece junto a Nuevo chat, cuando el espacio de trabajo del agente activo está respaldado por Git.

Los agentes de programación también pueden llamar a `spawn_task` cuando detectan trabajo de seguimiento confirmado fuera de la tarea actual. La interfaz de control muestra una sugerencia sin iniciar nada, mientras que una TUI respaldada por el Gateway muestra un mensaje interactivo con las mismas acciones. Seleccionar **Iniciar en un árbol de trabajo** crea un árbol de trabajo nuevo propiedad de la sesión a partir del proyecto sugerido y envía la instrucción autocontenida como su primer turno; descartar la sugerencia deja el repositorio intacto. Las sugerencias y sus identificadores son efímeros y no sobreviven a un reinicio del Gateway.

OpenClaw expone estas herramientas únicamente a las sesiones de operador con una interfaz del Gateway que permita actuar. Las sesiones de canal y las sesiones de TUI locales o integradas no las reciben hasta que esas superficies dispongan de un contrato portátil de acciones de tarea tipadas.

El árbol de trabajo administrado resultante pertenece a la sesión y cada ejecución del agente en esa sesión utiliza su checkout. Cuando el espacio de trabajo es un subdirectorio del repositorio, el árbol de trabajo se ancla en la raíz del repositorio y la sesión se ejecuta desde el subdirectorio correspondiente dentro de él. La creación del árbol de trabajo de sesión utiliza el ámbito `operator.write` del método, pero los hooks de checkout del repositorio y el paso `.openclaw/worktree-setup.sh` se ejecutan únicamente para los llamadores `operator.admin`, porque ejecutan código del repositorio; el aprovisionamiento de `.worktreeinclude` sigue aplicándose a todos los llamadores. Al eliminar la sesión, el árbol de trabajo se elimina únicamente cuando puede hacerse sin pérdida. Los árboles de trabajo con cambios o las ramas con commits sin enviar permanecen disponibles; la limpieza horaria genera instantáneas de los árboles de trabajo de sesión tras 7 días de inactividad y considera la actividad reciente de la sesión como actividad del árbol de trabajo. Los árboles de trabajo eliminados pueden restaurarse desde sus instantáneas como se describe a continuación.

`sessions.create` puede incluir un `cwd` absoluto para ejecutarse directamente en otra carpeta del Gateway, elegir el checkout de origen junto con `worktree: true` o establecer el directorio de trabajo de un nodo emparejado. Cada ruta de host explícita requiere `operator.admin`; la creación ordinaria de chats con árboles de trabajo sigue siendo `operator.write` y permanece anclada al espacio de trabajo configurado.

`sessions.create` también acepta `worktreeBaseRef` y `worktreeName` junto con `worktree: true` para elegir la referencia base y el nombre del árbol de trabajo (la rama pasa a ser `openclaw/<name>`); ambos permanecen en `operator.write`. El árbol de trabajo creado se devuelve en el resultado de creación y se conserva en la fila de la sesión como `worktree: { id, branch, repoRoot }`, de modo que las listas de sesiones pueden mostrar el checkout y la rama. Al eliminar una sesión, un checkout con cambios que se conserve se indica como `worktreePreserved` en lugar de dejarlo atrás silenciosamente.

## Instantáneas, limpieza y restauración

La eliminación crea primero un commit sintético que contiene los archivos rastreados y los archivos no rastreados y no ignorados, y después lo fija en `refs/openclaw/snapshots/<id>`. Los archivos ignorados nunca entran en la base de datos de objetos del repositorio. OpenClaw almacena únicamente los archivos ignorados que aprovisionó realmente en filas fragmentadas de la base de datos de estado compartida; el conjunto de rutas registrado sigue siendo la fuente de autoridad incluso si `.worktreeinclude` cambia o desaparece posteriormente. La restauración lee esos bytes de la instantánea inmutable y vuelve a aplicar sus modos completos. La limpieza automática conserva un árbol de trabajo activo cuando ya no es posible generar de forma segura la instantánea de una ruta registrada. Si falla la creación de la instantánea, la eliminación se detiene. Una eliminación forzada explícita puede continuar sin una instantánea.

OpenClaw aplica estas reglas de limpieza:

- Al finalizar la ejecución, elimina un árbol de trabajo únicamente cuando `git status --porcelain` está vacío y `git log HEAD --not --remotes --oneline` no encuentra commits sin enviar. De lo contrario, solo libera el bloqueo de actividad.
- La limpieza horaria genera instantáneas y elimina los árboles de trabajo desbloqueados propiedad de Workboard y de sesiones que lleven inactivos más de 7 días, incluso si tienen cambios. Los árboles de trabajo manuales nunca se eliminan automáticamente.
- Los registros de instantáneas pueden restaurarse durante 30 días. Después, la limpieza elimina la referencia de la instantánea y la fila del registro.
- El bloqueo de un proceso activo de OpenClaw y cualquier bloqueo de árbol de trabajo de git externo o no reconocido protegen un árbol de trabajo frente a la recolección de basura.

La restauración vuelve a crear `openclaw/<name>` en el commit original anterior a la instantánea y, después, reconstruye las diferencias de la instantánea como modificaciones no preparadas y archivos no rastreados. Esto mantiene el commit sintético de la instantánea fuera del historial de la rama. La referencia de la instantánea permanece registrada como procedencia.

## CLI

```bash
openclaw worktrees list [--json]
openclaw worktrees create <repo-root> [--name <name>] [--base-ref <ref>] [--json]
openclaw worktrees remove <id> [--force] [--json]
openclaw worktrees restore <id> [--json]
openclaw worktrees gc [--json]
```

La página **Árboles de trabajo** de la interfaz de control, en Configuración, proporciona las mismas acciones, además de la creación mediante un selector de rama base; muestra el propietario de cada árbol de trabajo (manual, Workboard o la sesión propietaria con un enlace a su chat) y ofrece un reintento forzado cuando una eliminación informa de un error de instantánea.

## Métodos del Gateway

| Método               | Propósito                                                                 |
| -------------------- | ----------------------------------------------------------------------- |
| `worktrees.list`     | Enumerar los registros de árboles de trabajo activos y restaurables.                            |
| `worktrees.branches` | Enumerar las ramas locales y remotas de un repositorio para los selectores de referencias base.    |
| `worktrees.create`   | Crear o reutilizar un árbol de trabajo administrado con nombre.                               |
| `worktrees.remove`   | Generar una instantánea y eliminar un árbol de trabajo. Las eliminaciones forzadas indican `snapshotError`. |
| `worktrees.restore`  | Restaurar un árbol de trabajo eliminado desde su instantánea.                           |
| `worktrees.gc`       | Ejecutar ahora la limpieza por inactividad, orfandad y retención.                            |

`worktrees.list` requiere `operator.read`, y los métodos que realizan cambios requieren `operator.admin`. `worktrees.branches` necesita `operator.write` para los espacios de trabajo de agentes configurados, mientras que cualquier otra ruta de host requiere `operator.admin` (de acuerdo con el requisito de cwd `sessions.create`). Solo lee las referencias existentes y nunca realiza una obtención, y las ramas que solo existen en remoto se devuelven con el remoto incluido (`origin/feature-a`), de modo que cada nombre devuelto pueda resolverse como referencia base. Nueva sesión también puede solicitar a este método un estado de repositorio tipado; un directorio normal o un checkout no disponible no devuelve ninguna rama, en lugar de obligar a la interfaz a deducir la compatibilidad con Git a partir de una cadena de error.

## Espacios de trabajo de Workboard

El [Plugin Workboard](/es/plugins/workboard) incluido puede materializar el espacio de trabajo de una tarjeta como un árbol de trabajo administrado:

```json
{
  "kind": "worktree",
  "path": "/absolute/path/to/source-checkout",
  "branch": "main"
}
```

`path` identifica el checkout de git de origen. `branch` es opcional y se convierte en la referencia base. Para un llamador con acceso completo al host, Workboard crea o reutiliza `wb-<card-id>`, ejecuta el subagente con el checkout administrado como directorio de trabajo y vuelve a escribir en la tarjeta la ruta y la rama resueltas. Los clientes del Gateway necesitan `operator.admin` para la materialización con acceso completo al host. Al finalizar la ejecución, Workboard elimina el checkout únicamente cuando se demuestra que no habrá pérdidas; el trabajo con cambios o los commits sin enviar permanecen disponibles.

Para un llamador limitado al espacio de trabajo, `path` y la raíz del repositorio deben coincidir exactamente con el espacio de trabajo del agente de destino. Workboard se ejecuta entonces directamente en ese directorio y registra un espacio de trabajo de directorio en lugar de materializar en el host un árbol de trabajo administrado. El destino debe utilizar un sandbox de Docker escribible y no compartido para el mismo espacio de trabajo, el hash de su contenedor activo debe coincidir con los montajes y la política solicitados, y no debe exponer ejecución elevada, control del host, sesiones para todo el host, ejecución persistente en el host o nodo, ni herramientas de plugins y MCP sin clasificar. Si la política de destino o el contenedor activo tienen un alcance más amplio, el envío deja la tarjeta sin reclamar e informa del estado incompatible.
