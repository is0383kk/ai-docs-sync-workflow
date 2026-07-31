---
read_when:
    - Quiere una interfaz de terminal para el Gateway (apta para acceso remoto)
    - Se desea pasar la URL, el token y la sesión desde scripts
    - Quiere ejecutar la TUI en modo integrado local sin un Gateway
    - Quiere usar openclaw chat u openclaw tui --local
summary: Referencia de la CLI para `openclaw tui` (interfaz de usuario de terminal respaldada por el Gateway o integrada localmente)
title: TUI
x-i18n:
    generated_at: "2026-07-26T04:34:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5406f25bbd22c64867296c15112fafcaf8e1580c759e5fdc81fccfb62ae1e318
    source_path: cli/tui.md
    workflow: 16
---

# `openclaw tui`

Abra la interfaz de terminal conectada al Gateway o ejecútela en modo local
integrado.

Guía relacionada: [TUI](/es/web/tui)

## Opciones

| Indicador                    | Valor predeterminado                      | Descripción                                                                        |
| ---------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `--local`                    | `false`                                   | Ejecuta el entorno de ejecución local integrado del agente en lugar de un Gateway. |
| `--url <url>`                | `gateway.remote.url` de la configuración | URL WebSocket del Gateway.                                                         |
| `--token <token>`            | (ninguno)                                 | Token del Gateway, si es necesario.                                                |
| `--password <pass>`          | (ninguna)                                 | Contraseña del Gateway, si es necesaria.                                           |
| `--tls-fingerprint <sha256>` | `gateway.remote.tlsFingerprint`           | Huella digital esperada del certificado TLS para un Gateway `wss://` fijado.       |
| `--session <key>`            | `main` (o `global` cuando el ámbito es global) | Clave de sesión. Dentro del espacio de trabajo de un agente, selecciona automáticamente ese agente, salvo que tenga un prefijo. |
| `--deliver`                  | `false`                                   | Entrega las respuestas del asistente mediante los canales configurados.            |
| `--thinking <level>`         | (valor predeterminado del modelo)         | Anulación del nivel de razonamiento.                                                |
| `--message <text>`           | (ninguno)                                 | Envía un mensaje inicial tras conectarse.                                           |
| `--timeout-ms <ms>`          | `agents.defaults.timeoutSeconds`          | Tiempo de espera del agente. Los valores no válidos registran una advertencia y se ignoran. |
| `--history-limit <n>`        | `200`                                     | Entradas del historial que se cargarán al conectarse.                               |

Los alias `openclaw chat` y `openclaw terminal` invocan este comando con
`--local` implícito.

## Notas

- `--local` no se puede combinar con `--url`, `--token`, `--password` ni `--tls-fingerprint`.
- `tui` resuelve las SecretRefs de autenticación del Gateway configuradas para la autenticación mediante token o contraseña
  cuando es posible (proveedores `env`/`file`/`exec`).
- Sin una URL ni un puerto explícitos, `tui` utiliza el puerto local activo del Gateway
  registrado por el Gateway en ejecución. La configuración explícita de `--url`, `OPENCLAW_GATEWAY_URL`,
  `OPENCLAW_GATEWAY_PORT` y del Gateway remoto conserva la prioridad.
- Cuando se inicia desde un directorio configurado como espacio de trabajo de un agente, la TUI selecciona automáticamente
  ese agente como valor predeterminado de la clave de sesión (salvo que `--session` sea explícitamente
  `agent:<id>:...`).
- El modo local utiliza directamente el entorno de ejecución integrado del agente. La mayoría de las herramientas locales funcionan,
  pero las funciones exclusivas del Gateway no están disponibles.
- El modo local añade `/auth [provider]` a la superficie de comandos de la TUI.
- Las restricciones de aprobación de los plugins siguen aplicándose en modo local: las herramientas que requieren aprobación
  solicitan una decisión en el terminal; nada se aprueba automáticamente de forma silenciosa.
- Los [objetivos](/es/tools/goal) de la sesión aparecen en el pie de página y se pueden gestionar con
  `/goal`.

## Ejemplos

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "Compara mi configuración con la documentación y dime qué debo corregir"
# cuando se ejecuta dentro del espacio de trabajo de un agente, deduce automáticamente ese agente
openclaw tui --session bugfix
```

## Bucle de reparación de la configuración

Utilice el modo local para que el agente integrado inspeccione la configuración actual, la compare
con la documentación y ayude a repararla desde el mismo terminal.

Si `openclaw config validate` ya está fallando, ejecute primero `openclaw configure` o
`openclaw doctor --fix`; `openclaw chat` no omite la
protección contra configuraciones no válidas.

```bash
openclaw chat
```

A continuación, dentro de la TUI:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

Aplique correcciones específicas con `openclaw config set` o `openclaw configure` y, a continuación,
vuelva a ejecutar `openclaw config validate`. Consulte [TUI](/es/web/tui) y
[Configuración](/es/cli/config).

## Relacionado

- [Referencia de la CLI](/es/cli)
- [TUI](/es/web/tui)
- [Objetivo](/es/tools/goal)
