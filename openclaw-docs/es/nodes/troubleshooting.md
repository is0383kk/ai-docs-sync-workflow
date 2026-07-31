---
read_when:
    - Node está conectado, pero las herramientas de cámara, canvas, pantalla y ejecución fallan
    - Necesita comprender el modelo mental de emparejamiento de nodos frente al de aprobaciones
summary: Solucionar problemas de emparejamiento de Node, requisitos de primer plano, permisos y fallos de herramientas
title: Solución de problemas de Node
x-i18n:
    generated_at: "2026-07-26T04:46:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

Use esta página cuando un nodo sea visible en el estado, pero las herramientas del nodo fallen.

## Secuencia de comandos

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

A continuación, ejecute las comprobaciones específicas del nodo:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

Indicadores de funcionamiento correcto:

- El nodo está conectado y emparejado para el rol `node`.
- `nodes describe` incluye la capacidad que se está invocando.
- Las aprobaciones de ejecución muestran el modo o la lista de permitidos esperados.

## Requisitos de primer plano

`canvas.*`, `camera.*` y `screen.*` solo funcionan en primer plano en nodos iOS/Android.

Comprobación y solución rápidas:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

Si aparece `NODE_BACKGROUND_UNAVAILABLE`, lleve la aplicación del nodo al primer plano y vuelva a intentarlo.

## Matriz de permisos

| Capacidad                    | iOS                                     | Android                                      | Aplicación de nodo para macOS    | Código de error habitual                       |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | -------------------------------- | --------------------------------------------- |
| `camera.snap`, `camera.clip` | Cámara (+ micrófono para el audio del clip) | Cámara (+ micrófono para el audio del clip) | Cámara (+ micrófono para el audio del clip) | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | Grabación de pantalla (+ micrófono opcional) | Solicitud de captura de pantalla (+ micrófono opcional) | Grabación de pantalla | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | No disponible                           | No disponible                                | Accesibilidad + Grabación de pantalla | `COMPUTER_DISABLED`, `ACCESSIBILITY_REQUIRED` |
| `location.get`               | While Using o Always (depende del modo) | Ubicación en primer o segundo plano según el modo | Permiso de ubicación             | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | No disponible (ruta del host del nodo)  | No disponible (ruta del host del nodo)       | Se requieren aprobaciones de ejecución | `SYSTEM_RUN_DENIED`                           |

## Emparejamiento frente a aprobaciones

Tres controles independientes determinan si un comando de nodo se ejecuta correctamente:

1. **Emparejamiento del dispositivo**: ¿puede este nodo conectarse al Gateway?
2. **Política de comandos de nodo del Gateway**: ¿permiten `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` y los valores predeterminados de la plataforma el identificador del comando RPC?
3. **Aprobaciones de ejecución**: ¿puede este nodo ejecutar localmente un comando de shell específico?

El emparejamiento de nodos es un control de identidad y confianza, no una interfaz de aprobación por comando. Para `system.run`, la política por nodo reside en el archivo de aprobaciones de ejecución de ese nodo (`openclaw approvals get --node ...`), no en el registro de emparejamiento del Gateway.

Comprobaciones rápidas:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- Falta el emparejamiento: apruebe primero el dispositivo del nodo.
- A `nodes describe` le falta un comando: compruebe la política de comandos de nodo del Gateway y si el nodo declaró realmente ese comando al conectarse.
- El emparejamiento es correcto, pero `system.run` falla: corrija las aprobaciones de ejecución o la lista de permitidos en ese nodo.

Para las ejecuciones de `host=node` respaldadas por aprobación, el Gateway también vincula la ejecución al `systemRunPlan` canónico preparado. Si posteriormente un invocador modifica el comando, el directorio de trabajo o los metadatos de la sesión antes de reenviar la ejecución aprobada, el Gateway rechaza la ejecución por discrepancia con la aprobación en lugar de confiar en la carga útil modificada.

## Códigos de error habituales de los nodos

| Código                                 | Significado                                                                                                                                                                             |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | La aplicación está en segundo plano; llévela al primer plano.                                                                                                                           |
| `CAMERA_DISABLED`                      | El selector de la cámara está desactivado en la configuración del nodo.                                                                                                                 |
| `*_PERMISSION_REQUIRED`                | Falta el permiso del sistema operativo o se ha denegado.                                                                                                                                |
| `LOCATION_DISABLED`                    | El modo de ubicación está desactivado.                                                                                                                                                  |
| `LOCATION_PERMISSION_REQUIRED`         | No se ha concedido el modo de ubicación solicitado.                                                                                                                                     |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | La aplicación está en segundo plano, pero solo existe el permiso While Using.                                                                                                            |
| `COMPUTER_DISABLED`                    | Active **Allow Computer Control** en la aplicación para macOS y, a continuación, apruebe la actualización del emparejamiento.                                                          |
| `ACCESSIBILITY_REQUIRED`               | Conceda Accessibility al paquete actual de la aplicación OpenClaw en macOS System Settings.                                                                                             |
| `SYSTEM_RUN_DENIED: approval required` | La solicitud de ejecución requiere aprobación explícita.                                                                                                                               |
| `SYSTEM_RUN_DENIED: allowlist miss`    | El modo de lista de permitidos bloqueó el comando. En los hosts de nodo Windows, las formas de envoltorio de shell como `cmd.exe /c ...` se consideran ausentes de la lista de permitidos en ese modo, salvo que se aprueben mediante el flujo de solicitud. |

## Bucle de recuperación rápida

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

Si el problema persiste:

- Vuelva a aprobar el emparejamiento del dispositivo.
- Vuelva a abrir la aplicación del nodo en primer plano.
- Vuelva a conceder los permisos del sistema operativo.
- Vuelva a crear o ajuste la política de aprobación de ejecución.

Para el control del equipo, verifique también que un agente con capacidad de visión exponga la herramienta `computer`, que `screen.snapshot` se ejecute correctamente con el permiso Grabación de pantalla y que `/phone status` muestre la autorización temporal o persistente del Gateway prevista. Una entrada `gateway.nodes.commands.deny` siempre prevalece sobre `gateway.nodes.commands.allow`.

## Temas relacionados

- [Descripción general de los nodos](/es/nodes)
- [Nodos de cámara](/es/nodes/camera)
- [Comando de ubicación](/es/nodes/location-command)
- [Uso del equipo](/es/nodes/computer-use)
- [Aprobaciones de ejecución](/es/tools/exec-approvals)
- [Emparejamiento del Gateway](/es/gateway/pairing)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
- [Solución de problemas de canales](/es/channels/troubleshooting)
