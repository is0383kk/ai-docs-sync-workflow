---
read_when:
    - Quieres editar las aprobaciones de ejecución desde la CLI
    - Es necesario gestionar las listas de permitidos en los hosts del Gateway o de los nodos.
    - Necesita listar o resolver una aprobación pendiente sin una interfaz de chat
summary: Referencia de la CLI para `openclaw approvals` y `openclaw exec-policy`
title: Aprobaciones
x-i18n:
    generated_at: "2026-07-26T04:32:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

Gestiona las aprobaciones de ejecución para el **host local**, el **host del Gateway** o un **host de nodo**. Si no se especifica ningún indicador de destino, los comandos leen o escriben el archivo de aprobaciones local en el disco. Usa `--gateway` para seleccionar el Gateway o `--node <id|name|ip>` para seleccionar un nodo específico.

Alias: `openclaw exec-approvals`

Relacionado: [Aprobaciones de ejecución](/es/tools/exec-approvals), [Nodos](/es/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` es el comando práctico **solo local** que mantiene sincronizados en un solo paso la configuración solicitada de `tools.exec.*` y el archivo de aprobaciones del host local:

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

Los ajustes preestablecidos (`yolo`, `cautious`, `deny-all`) aplican conjuntamente `host`, `security`, `ask` y `askFallback`. `set` aplica únicamente los indicadores especificados; se valida cada valor aceptado (`--host auto|sandbox|gateway|node`, `--security deny|allowlist|full`, `--ask off|on-miss|always`, `--ask-fallback deny|allowlist|full`).

Ámbito:

- Actualiza conjuntamente el archivo de configuración local y el archivo de aprobaciones local; no envía la política al Gateway ni a un host de nodo.
- `--host node` se rechaza: las aprobaciones de ejecución del nodo se obtienen del nodo durante la ejecución, por lo que el `exec-policy` local no puede sincronizarlas. Usa `openclaw approvals set --node <id|name|ip>` en su lugar.
- `exec-policy show` marca los ámbitos de `host=node` como gestionados por el nodo durante la ejecución, en lugar de derivar una política efectiva del archivo de aprobaciones local.

Para las aprobaciones de hosts remotos, usa directamente `openclaw approvals set --gateway` o `openclaw approvals set --node <id|name|ip>`.

## Comandos habituales

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` muestra la política de ejecución efectiva para el destino: la política solicitada de `tools.exec`, la política del archivo de aprobaciones del host y el resultado efectivo combinado. Los nodos con una política nativa del host, como la aplicación complementaria de Windows, muestran esa política directamente en lugar de aplicar los cálculos de la política del archivo de aprobaciones de OpenClaw.

En los nodos respaldados por archivos, la vista combinada requiere una instantánea de la política resuelta por el host. Los nodos más antiguos muestran la política efectiva como no disponible, en lugar de suponer que la política solicitada del Gateway también se aplica en el host.

<Note>
No se incluyen las anulaciones de `/exec` por sesión. Ejecuta `/exec` en la sesión correspondiente para consultar sus valores predeterminados actuales.
</Note>

Precedencia:

- El archivo de aprobaciones del host es la fuente de verdad aplicable.
- La política solicitada de `tools.exec` puede restringir o ampliar la intención, pero el resultado efectivo se deriva de las reglas del host.
- `--node` combina el archivo de aprobaciones del host de nodo con la política de `tools.exec` del Gateway (ambos se aplican durante la ejecución).
- Si la configuración del Gateway no está disponible, la CLI recurre a la instantánea de aprobaciones del nodo e indica que no se pudo calcular la política final durante la ejecución.

## Aprobaciones pendientes

Enumera las aprobaciones pendientes de ejecución, plugins y agentes del sistema de OpenClaw del Gateway:

```bash
openclaw approvals pending
openclaw approvals pending --json
```

La enumeración completa y el flujo correspondiente de `resolve` para todo el operador usan `operator.admin`, porque, de lo contrario, los registros de aprobación conservan el filtrado por solicitante y revisor. La resolución también solicita el ámbito específico `operator.approvals`. La concesión estándar del operador de la CLI incluye ambos ámbitos; un cliente restringido de terceros no debe solicitar privilegios de administración únicamente para emular este comando.

La salida legible muestra el tipo de aprobación, la atribución del agente y la sesión, la antigüedad de la solicitud, el tiempo restante hasta su vencimiento, un comando o resumen abreviado y un token de identificador `id64_<base64url>` independiente del shell. Después de la tabla compacta siempre aparece un bloque `Full request text` con todos los tokens completos y una solicitud escapada sin pérdida, para que la abreviación por el ancho del terminal no pueda ocultar un sufijo ni el token necesario para la resolución. Copia el token completo en `resolve`. Los caracteres de terminal no seguros de otros campos se muestran como secuencias de escape Unicode visibles. La salida JSON devuelve entradas normalizadas en `approvals` y conserva los valores sin procesar originales de `id`, `summary`, `createdAtMs` y `expiresAtMs` para los scripts; `resolve` sigue aceptando los identificadores sin procesar, salvo que usen el prefijo reservado de token de visualización `id64_`.

Si un valor `id64_` proporcionado coincide tanto con un identificador literal sin procesar como con el token de visualización decodificado de otra aprobación, la CLI lo rechaza por ambiguo, en lugar de arriesgarse a resolver la solicitud incorrecta.

Resuelve una aprobación mediante su identificador completo:

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "No se esperaba durante el mantenimiento"
```

La CLI lee el registro de aprobación unificado para seleccionar su tipo, comprueba la decisión solicitada con respecto a las decisiones permitidas del registro y, a continuación, llama al solucionador unificado. Una primera decisión correcta termina con `0`. Repetir la decisión registrada también termina con `0` e informa de `already resolved (same decision)`. Una decisión contradictoria, una aprobación inexistente o vencida, o una decisión no disponible para ese tipo de aprobación muestra un error claro y termina con un código distinto de cero.

`--reason` añade una nota local a la confirmación de la CLI. El registro de aprobación actual del Gateway no tiene ningún campo de texto libre para el motivo de la resolución, por lo que esta nota no se conserva ni se envía a otras superficies de aprobación.

## Sustituir las aprobaciones desde un archivo

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` acepta JSON5, no solo JSON estricto. Usa `--file` o `--stdin`, pero no ambos.

Los nodos Windows nativos del host usan su propia estructura de política:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

La CLI lee primero el hash actual del nodo y lo envía con la actualización, de modo que las ediciones locales simultáneas se rechacen en lugar de sobrescribirse. `rules` es obligatorio porque esta operación sustituye la lista completa de reglas del nodo; `defaultAction` es opcional. Un nodo que indique que su política nativa está deshabilitada no puede configurarse de forma remota; primero habilita o configura la política en ese host. Las políticas nativas del host no admiten los asistentes `allowlist add|remove`.

## Ejemplo de «No preguntar nunca» / YOLO

Establece los valores predeterminados de las aprobaciones del host en `full` + `off` para un host que nunca deba detenerse por aprobaciones de ejecución:

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

Para los nodos que exponen un archivo de aprobaciones de OpenClaw, usa el mismo cuerpo con `openclaw approvals set --node <id|name|ip> --stdin`. Los nodos nativos del host requieren la estructura específica del propietario mostrada anteriormente.

Esto cambia únicamente el **archivo de aprobaciones del host**. Para mantener alineada la política solicitada de OpenClaw, configura también:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

`tools.exec.host=gateway` es explícito aquí porque `host=auto` sigue significando «usar el entorno aislado cuando esté disponible; de lo contrario, usar el Gateway»: YOLO se refiere a las aprobaciones, no al enrutamiento. Usa `gateway` (o `/exec host=gateway`) cuando se quiera ejecutar en el host incluso si hay un entorno aislado configurado.

Si se omite `askFallback`, el valor predeterminado es `deny`. Configura `askFallback: "full"` explícitamente al actualizar un host sin interfaz de usuario que deba conservar el comportamiento de no preguntar nunca.

Atajo local para la misma intención, solo en la máquina local:

```bash
openclaw exec-policy preset yolo
```

## Asistentes de la lista de permitidos

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## Opciones habituales

`get`, `set` y `allowlist add|remove` admiten:

- `--node <id|name|ip>` (resuelve identificadores, nombres, direcciones IP o prefijos de identificador; usa el mismo solucionador que `openclaw nodes`)
- `--gateway`
- opciones compartidas de RPC de nodo: `--url`, `--token`, `--timeout`, `--json`

Si no se especifica ningún indicador de destino, se usa el archivo de aprobaciones local del disco.

`allowlist add|remove` también admite `--agent <id>` (el valor predeterminado es `"*"`, que se aplica a todos los agentes).

`pending` y `resolve` siempre usan el Gateway porque las solicitudes pendientes forman parte del estado activo del Gateway. Admiten las opciones compartidas de conexión al Gateway `--url`, `--token` y `--timeout`; `pending` también admite `--json`.

## Notas

- El host de nodo debe anunciar `system.execApprovals.get/set` (aplicación para macOS, host de nodo sin interfaz gráfica o aplicación complementaria de Windows).
- Los archivos de aprobaciones se almacenan por host en el directorio de estado de OpenClaw: `$OPENCLAW_STATE_DIR/exec-approvals.json`, o `~/.openclaw/exec-approvals.json` cuando la variable no está definida.

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Aprobaciones de ejecución](/es/tools/exec-approvals)
