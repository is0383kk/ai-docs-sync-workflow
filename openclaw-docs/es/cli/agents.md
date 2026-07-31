---
read_when:
    - Quieres varios agentes aislados (espacios de trabajo + enrutamiento + autenticación)
summary: Referencia de la CLI para `openclaw agents` (listar/añadir/eliminar/vinculaciones/vincular/desvincular/establecer identidad)
title: Agentes
x-i18n:
    generated_at: "2026-07-26T04:33:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76a2e50462f6a52760dcb639405ed5f23857f2fa429469281e3acfa1eb61e974
    source_path: cli/agents.md
    workflow: 16
---

# `openclaw agents`

Gestiona agentes aislados (espacios de trabajo + autenticación + enrutamiento). Ejecutar `openclaw agents` sin un subcomando equivale a `openclaw agents list`.

Relacionado:

- [Enrutamiento multiagente](/es/concepts/multi-agent)
- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
- [Configuración de Skills](/es/tools/skills-config): configuración de la visibilidad de las habilidades.

## Ejemplos

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:*
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## Superficie de comandos

### `agents list`

Opciones: `--json`, `--bindings` (incluye las reglas de enrutamiento completas, no solo los recuentos/resúmenes por agente).

### `agents add [name]`

Opciones: `--workspace <dir>`, `--model <id>`, `--agent-dir <dir>`, `--bind <channel[:accountId]>` (se puede repetir), `--non-interactive`, `--json`.

- Proporcionar cualquier marca de adición explícita cambia el comando a la ruta no interactiva.
- El modo no interactivo requiere tanto un nombre de agente como `--workspace`.
- `main` está reservado y no se puede usar como id del nuevo agente.
- El modo interactivo inicializa la autenticación copiando únicamente credenciales estáticas portátiles (perfiles `api_key` y perfiles `token` estáticos), salvo que una credencial deshabilite la copia mediante `copyToAgents: false`; los perfiles de tokens de actualización OAuth no se copian, salvo que un proveedor habilite la copia mediante `copyToAgents: true`. Si no se realiza una copia, OAuth permanece disponible únicamente mediante la herencia de lectura desde el almacén real del agente `main`. Si el agente predeterminado configurado no es `main`, se debe iniciar sesión por separado para los perfiles OAuth del nuevo agente.

### `agents bindings`

Opciones: `--agent <id>`, `--json`.

### `agents bind`

Opciones: `--agent <id>` (de forma predeterminada, el agente predeterminado actual), `--bind <channel[:accountId]>` (se puede repetir), `--json`.

### `agents unbind`

Opciones: `--agent <id>` (de forma predeterminada, el agente predeterminado actual), `--bind <channel[:accountId]>` (se puede repetir), `--all`, `--json`. Acepta `--all` o uno o varios valores `--bind`, pero no ambos.

### `agents set-identity`

Opciones: `--agent <id>`, `--workspace <dir>`, `--identity-file <path>`, `--from-identity`, `--name <name>`, `--theme <theme>`, `--emoji <emoji>`, `--avatar <value>`, `--json`. Consulte [Establecer la identidad](#set-identity) a continuación.

### `agents delete <id>`

Opciones: `--force`, `--json`.

- `main` no se puede eliminar.
- Sin `--force`, se requiere confirmación interactiva (falla en una sesión sin TTY; vuelva a ejecutar con `--force`).
- Los directorios del espacio de trabajo, del estado del agente y de las transcripciones de las sesiones se trasladan a la papelera, no se eliminan de forma permanente. Si la papelera no está disponible, la eliminación de la configuración del agente se completa igualmente y se notifican las rutas que requieren limpieza manual.
- Cuando el Gateway está accesible, la eliminación se enruta a través del Gateway para que la limpieza de la configuración y del almacén de sesiones comparta el mismo escritor que el tráfico en tiempo de ejecución. Si el Gateway no está accesible, la CLI recurre a la ruta local sin conexión.
- Si el espacio de trabajo de otro agente tiene la misma ruta, se encuentra dentro de este espacio de trabajo o contiene este espacio de trabajo, el espacio de trabajo se conserva y `--json` informa de `workspaceRetained`, `workspaceRetainedReason` y `workspaceSharedWith`.

## Vinculaciones de enrutamiento

Utilice vinculaciones de enrutamiento para fijar el tráfico entrante de un canal a un agente específico.

Si también se desean diferentes Skills visibles por agente, configure `agents.defaults.skills` y `agents.entries.*.skills` en `openclaw.json`. Consulte [Configuración de Skills](/es/tools/skills-config) y [Referencia de configuración](/es/gateway/config-agents#agentsdefaultsskills).

Enumerar vinculaciones:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

Añadir vinculaciones:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

También se pueden añadir vinculaciones al crear un agente:

```bash
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:* --bind discord:*
```

Si se omite `accountId` (`--bind <channel>`), OpenClaw lo resuelve mediante los enlaces de configuración del Plugin, la vinculación forzada de la cuenta o el número de cuentas configuradas del canal.

Si se omite `--agent` para `bind` o `unbind`, OpenClaw usa como destino el agente predeterminado actual.

### Formato de `--bind`

| Formato                       | Significado                                                                                        |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| `--bind <channel>:*`         | Coincide con todas las cuentas del canal.                                                          |
| `--bind <channel>:<account>` | Coincide con una cuenta.                                                                           |
| `--bind <channel>`           | Coincide únicamente con la cuenta predeterminada, salvo que la CLI pueda resolver de forma segura un ámbito de cuenta específico del Plugin. |

### Comportamiento del ámbito de vinculación

- Una vinculación almacenada sin `accountId` coincide únicamente con la cuenta predeterminada del canal.
- `accountId: "*"` es la alternativa para todo el canal (todas las cuentas) y es menos específica que una vinculación de cuenta explícita.
- Si el mismo agente ya tiene una vinculación de canal coincidente sin `accountId` y posteriormente se vincula con un `accountId` explícito o resuelto, OpenClaw actualiza esa vinculación existente directamente en lugar de añadir un duplicado.

Ejemplos:

```bash
# coincidir con todas las cuentas del canal
openclaw agents bind --agent work --bind telegram:*

# coincidir con una cuenta específica
openclaw agents bind --agent work --bind telegram:ops

# vinculación inicial solo al canal
openclaw agents bind --agent work --bind telegram

# actualización posterior a una vinculación con ámbito de cuenta
openclaw agents bind --agent work --bind telegram:alerts
```

Tras la actualización, el enrutamiento de esa vinculación queda limitado a `telegram:alerts`. Si también se desea el enrutamiento de la cuenta predeterminada, añádalo explícitamente (por ejemplo, `--bind telegram:default`).

Eliminar vinculaciones:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## Archivos de identidad

Cada espacio de trabajo de agente puede incluir un archivo `IDENTITY.md` en la raíz del espacio de trabajo:

- Ruta de ejemplo: `~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity` lee desde la raíz del espacio de trabajo (o desde un `--identity-file` explícito).

Las rutas de los avatares se resuelven con respecto a la raíz del espacio de trabajo y no pueden salir de ella, ni siquiera mediante un enlace simbólico.

## Establecer la identidad

`set-identity` escribe campos en `agents.entries.*.identity`: `name`, `theme`, `emoji`, `avatar` (ruta relativa al espacio de trabajo, URL http(s) o URI de datos).

- `--agent` o `--workspace` selecciona el agente de destino. Si `--workspace` coincide con más de un agente, el comando falla y solicita que se proporcione `--agent`.
- Los archivos locales de imagen de avatar con rutas relativas al espacio de trabajo están limitados a 2 MB. Las URL HTTP(S) y los URI `data:` no se comprueban con respecto al límite de tamaño de los archivos locales.
- Cuando no se proporcionan campos de identidad explícitos, el comando lee los datos de identidad de `IDENTITY.md`.

Cargar desde `IDENTITY.md`:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

Sobrescribir campos explícitamente:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

Ejemplo de configuración:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Enrutamiento multiagente](/es/concepts/multi-agent)
- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
