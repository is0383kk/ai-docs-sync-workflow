---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Por qué se bloquea una herramienta: entorno de ejecución del sandbox, política de permisos y denegaciones de herramientas, y controles de ejecución con privilegios elevados'
title: Sandbox frente a política de herramientas frente a privilegios elevados
x-i18n:
    generated_at: "2026-07-26T04:40:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw tiene tres controles relacionados pero diferentes:

1. **Entorno aislado** (`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`) decide **dónde se ejecutan las herramientas** (backend del entorno aislado o host).
2. **Política de herramientas** (`tools.*`, `tools.sandbox.tools.*`, `agents.entries.*.tools.*`) decide **qué herramientas están disponibles o permitidas**.
3. **Elevado** (`tools.elevated.*`, `agents.entries.*.tools.elevated.*`) es una **vía de escape exclusiva para exec** que permite ejecutar fuera del entorno aislado cuando se está en uno (`gateway` de forma predeterminada, o `node` cuando el destino de exec está configurado como `node`).

## Depuración rápida

Utilice el inspector para ver qué está haciendo _realmente_ OpenClaw:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Muestra:

- el modo, el ámbito y el acceso al espacio de trabajo efectivos del entorno aislado
- si la sesión se encuentra actualmente en un entorno aislado (principal frente a no principal)
- la política efectiva de herramientas permitidas y denegadas del entorno aislado (y si procede del agente, de la configuración global o de los valores predeterminados)
- las puertas de acceso al modo elevado y las rutas de claves para corregirlas

## Entorno aislado: dónde se ejecutan las herramientas

El aislamiento está controlado por `agents.defaults.sandbox.mode`:

- `"off"`: todo se ejecuta en el host.
- `"non-main"`: solo las sesiones no principales se ejecutan en un entorno aislado (una «sorpresa» habitual en grupos y canales).
- `"all"`: todo se ejecuta en un entorno aislado.

`agents.defaults.sandbox.workspaceAccess` controla qué puede ver el entorno aislado: `"none"`, `"ro"` o `"rw"`.

Consulte [Aislamiento](/es/gateway/sandboxing) para ver la matriz completa (ámbito, montajes del espacio de trabajo e imágenes).

### Montajes vinculados (comprobación rápida de seguridad)

- `docker.binds` _atraviesa_ el sistema de archivos del entorno aislado: todo lo que se monte será visible dentro del contenedor con el modo establecido (`:ro` o `:rw`).
- Si se omite el modo, el valor predeterminado es lectura y escritura; se recomienda `:ro` para código fuente y secretos.
- `scope: "shared"` ignora los montajes vinculados por agente (solo se aplican los globales).
- OpenClaw valida dos veces los orígenes de los montajes vinculados: primero en la ruta de origen normalizada y después de resolverla mediante el ancestro existente más profundo. Las evasiones mediante un enlace simbólico en un directorio superior no eluden las comprobaciones de rutas bloqueadas ni de raíces permitidas.
- Las rutas de hojas inexistentes también se comprueban de forma segura. Si `/workspace/alias-out/new-file` se resuelve mediante un directorio superior con enlace simbólico hacia una ruta bloqueada o fuera de las raíces permitidas configuradas, se rechaza el montaje vinculado.
- Vincular `/var/run/docker.sock` entrega, en la práctica, el control del host al entorno aislado; hágalo únicamente de forma intencionada.
- El acceso al espacio de trabajo (`workspaceAccess`) es independiente de los modos de los montajes vinculados.

Para consultar una configuración por agente con varias carpetas del host, modos de acceso y la habilitación de seguridad para fuentes externas, consulte [Varias carpetas para un agente](/es/gateway/sandboxing#multiple-folders-for-one-agent).

## Política de herramientas: qué herramientas existen o pueden invocarse

Importan dos capas:

- **Perfil de herramientas**: `tools.profile` y `agents.entries.*.tools.profile` (lista base de permitidas)
- **Perfil de herramientas del proveedor**: `tools.byProvider[provider].profile` y `agents.entries.*.tools.byProvider[provider].profile`
- **Política de herramientas global o por agente**: `tools.allow`/`tools.deny` y `agents.entries.*.tools.allow`/`agents.entries.*.tools.deny`
- **Política de herramientas del proveedor**: `tools.byProvider[provider].allow/deny` y `agents.entries.*.tools.byProvider[provider].allow/deny`
- **Política de herramientas del entorno aislado** (solo se aplica durante el aislamiento): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` y `agents.entries.*.tools.sandbox.tools.*`

Reglas generales:

- `deny` siempre prevalece.
- Si `allow` no está vacío, todo lo demás se considera bloqueado.
- La política de herramientas es el límite definitivo: `/exec` no puede anular la denegación de una herramienta `exec`.
- La política de herramientas filtra su disponibilidad por nombre; no inspecciona los efectos secundarios dentro de `exec`. Si se permite `exec`, denegar `write`, `edit` o `apply_patch` no convierte los comandos del shell en operaciones de solo lectura.
- `/exec` solo cambia los valores predeterminados de la sesión para remitentes autorizados; no concede acceso a herramientas.
- Las claves de herramientas del proveedor aceptan `provider` (por ejemplo, `google-antigravity`) o `provider/model` (por ejemplo, `openai/gpt-5.4`).
- Los registros del Gateway incluyen entradas de auditoría `agents/tool-policy` cuando un paso de la política de herramientas elimina herramientas o cuando una política de herramientas del entorno aislado bloquea una llamada. Utilice `openclaw logs` para ver la etiqueta de la regla, la clave de configuración y los nombres de las herramientas afectadas.

### Grupos de herramientas (abreviaturas)

Las políticas de herramientas (globales, de agente y del entorno aislado) admiten entradas `group:*` que se expanden a varias herramientas:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Grupos disponibles:

| Grupo              | Herramientas                                                                                                                                                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` se acepta como alias de `exec`)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | la mayoría de las herramientas integradas de OpenClaw (excluye las primitivas del sistema de archivos y de tiempo de ejecución `read`/`write`/`edit`/`apply_patch`/`exec`/`process`, `canvas` y los plugins de proveedores)                                                                                             |
| `group:plugins`    | todas las herramientas cargadas pertenecientes a plugins, incluidos los servidores MCP configurados expuestos mediante `bundle-mcp`                                                                                                                                                           |

Para los agentes de solo lectura, deniegue `group:runtime` además de las herramientas que modifican el sistema de archivos, a menos que la política del sistema de archivos del entorno aislado o un límite independiente del host imponga la restricción de solo lectura.

Para los servidores MCP en entornos aislados, la política de herramientas del entorno aislado constituye una segunda puerta de autorización. Si `mcp.servers` está configurado, pero los turnos en entornos aislados solo muestran herramientas integradas, añada `bundle-mcp`, `group:plugins` o un nombre o patrón global de herramienta MCP con prefijo de servidor, como `outlook__send_mail` o `outlook__*`, a `tools.sandbox.tools.alsoAllow`; después, reinicie o vuelva a cargar el Gateway y capture de nuevo la lista de herramientas. Los patrones globales de servidor utilizan el prefijo del servidor MCP seguro para proveedores: los caracteres que no sean `[A-Za-z0-9_-]` se convierten en `-`, los nombres que no comiencen por una letra reciben el prefijo `mcp-`, y los prefijos largos o duplicados pueden truncarse o recibir un sufijo.

`openclaw doctor` comprueba actualmente esta estructura para los servidores administrados por OpenClaw en `mcp.servers`. Los servidores MCP cargados desde manifiestos de plugins integrados o desde `.mcp.json` de Claude utilizan la misma puerta de acceso del entorno aislado, pero este diagnóstico aún no enumera esas fuentes; utilice las mismas entradas de la lista de permitidas si sus herramientas desaparecen en los turnos ejecutados en entornos aislados.

## Elevado: «ejecutar en el host» solo para exec

El modo elevado **no** concede herramientas adicionales; solo afecta a `exec`.

- Si se está en un entorno aislado, `/elevated on` (o `exec` con `elevated: true`) se ejecuta fuera de este (es posible que sigan siendo necesarias las aprobaciones).
- Utilice `/elevated full` para omitir las aprobaciones de exec durante la sesión.
- Si ya se está ejecutando directamente, el modo elevado no tiene ningún efecto práctico (aunque sigue sujeto a las puertas de acceso).
- El modo elevado **no** está limitado a una skill y **no** anula las reglas de autorización o denegación de herramientas.
- El modo elevado no concede anulaciones arbitrarias entre hosts desde `host=auto`; sigue las reglas normales del destino de exec y solo conserva `node` cuando el destino configurado o de la sesión ya es `node`.
- `/exec` es independiente del modo elevado. Solo ajusta los valores predeterminados de exec por sesión para remitentes autorizados.

Puertas de acceso:

- Habilitación: `tools.elevated.enabled` (y, opcionalmente, `agents.entries.*.tools.elevated.enabled`)
- Listas de remitentes permitidos: `tools.elevated.allowFrom.<provider>` (y, opcionalmente, `agents.entries.*.tools.elevated.allowFrom.<provider>`)

Consulte [Modo elevado](/es/tools/elevated).

## Soluciones habituales para el «confinamiento del entorno aislado»

### «La herramienta X está bloqueada por la política de herramientas del entorno aislado»

Claves para corregirlo (elija una):

- Deshabilitar el entorno aislado: `agents.defaults.sandbox.mode=off` (o `agents.entries.*.sandbox.mode=off` por agente)
- Permitir la herramienta dentro del entorno aislado:
  - quitarla de `tools.sandbox.tools.deny` (o `agents.entries.*.tools.sandbox.tools.deny` por agente)
  - o añadirla a `tools.sandbox.tools.allow` (o permitirla por agente)
- Comprobar la entrada `agents/tool-policy` en `openclaw logs`. Registra el modo del entorno aislado y si la regla de permiso o denegación bloqueó la herramienta.

### «Creía que esta era la sesión principal, ¿por qué está aislada?»

En el modo `"non-main"`, las claves de grupo/canal _no_ corresponden a la sesión principal. Usar la clave de la sesión principal (mostrada por `sandbox explain`) o cambiar el modo a `"off"`.

## Contenido relacionado

- [Entorno aislado](/es/gateway/sandboxing) -- referencia completa del entorno aislado (modos, ámbitos, backends e imágenes)
- [Entorno aislado y herramientas multiagente](/es/tools/multi-agent-sandbox-tools) -- anulaciones por agente y precedencia
- [Modo elevado](/es/tools/elevated)
