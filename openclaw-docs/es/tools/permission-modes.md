---
read_when:
    - Elegir auto, ask, allowlist, full o deny para los permisos de comandos
    - Configuración de aprobaciones revisadas por Codex Guardian mediante tools.exec.mode
    - Comparación de las aprobaciones de ejecución de OpenClaw con los permisos del arnés ACPX
summary: Modos de permisos para la ejecución en el host, las aprobaciones de Codex Guardian y las sesiones del entorno ACPX
title: Modos de permisos
x-i18n:
    generated_at: "2026-07-26T05:58:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f580e66508c1f69e868ed26a62d88a675f86a4d1ca738650dc5af82e967f3ac3
    source_path: tools/permission-modes.md
    workflow: 16
---

Los modos de permisos determinan cuánta autoridad tiene un agente antes de ejecutar comandos en el host, escribir archivos o solicitar acceso adicional a un entorno de ejecución de backend.

<Note>
  El modo de permisos es independiente de `tools.exec.host=auto`. `tools.exec.host`
  determina dónde se ejecuta un comando. `tools.exec.mode` determina cómo se
  aprueba la ejecución en el host.
</Note>

## Valor predeterminado recomendado

Utilice `auto` para agentes de programación que necesiten un acceso útil al host sin convertir cada incumplimiento en una solicitud a una persona:

```bash
openclaw config set tools.exec.mode auto
openclaw approvals get
openclaw gateway restart
```

A continuación, verifique la política efectiva:

```bash
openclaw exec-policy show
```

## Modos de ejecución en el host de OpenClaw

`tools.exec.mode` es la superficie de política normalizada para `exec` del host. Cada modo se resuelve en un par subyacente de `security` (grado de restricción de la lista de permitidos) y `ask` (solicitud en caso de incumplimiento):

| Modo        | security / ask          | Comportamiento                                                                                      | Utilizar cuando                                              |
| ----------- | ----------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `deny`      | `deny` / `off`          | Bloquea por completo la ejecución en el host.                                                       | No se permite ningún comando en el host.                     |
| `allowlist` | `allowlist` / `off`     | Ejecuta únicamente comandos incluidos en la lista de permitidos; deniega silenciosamente los demás. | Se dispone de un conjunto de comandos cuya seguridad se conoce. |
| `ask`       | `allowlist` / `on-miss` | Ejecuta las coincidencias de la lista de permitidos; consulta a una persona para las demás.          | Una persona debe revisar cada comando nuevo.                 |
| `auto`      | `allowlist` / `on-miss` | Ejecuta las coincidencias de la lista de permitidos; envía las demás a revisión automática antes de recurrir a la aprobación humana. | Las sesiones de programación necesitan un acceso práctico y protegido. |
| `full`      | `full` / `off`          | Ejecuta comandos en el host sin solicitudes.                                                       | Este host o sesión de confianza debe omitir los controles de aprobación. |

`ask` y `auto` comparten la misma configuración de lista de permitidos y solicitudes; `auto` habilita además el revisor automático nativo, que decide por sí mismo sobre los incumplimientos y solo los remite a la ruta configurada de aprobación humana cuando no puede aprobarlos de forma segura.

Para consultar la política completa de ejecución en el host, el archivo local de aprobaciones, el esquema de la lista de permitidos, los binarios seguros y el comportamiento de reenvío, consulte [Aprobaciones de ejecución](/es/tools/exec-approvals).

## Asignación de Codex Guardian

En las sesiones nativas del servidor de aplicaciones de Codex, `tools.exec.mode: "auto"` orienta Codex hacia aprobaciones revisadas por Guardian cuando los requisitos locales de Codex lo permiten. Valores resultantes habituales:

| Campo de Codex         | Valor habitual     |
| ---------------------- | ------------------ |
| `approvalPolicy`    | `on-request`      |
| `approvalsReviewer` | `auto_review`     |
| `sandbox`           | `workspace-write` |

El modo `auto` impone esta política sobre cualquier anulación configurada del entorno aislado o de las aprobaciones de Codex, por lo que no conserva combinaciones inseguras heredadas como `approvalPolicy: "never"` con `sandbox: "danger-full-access"`. `tools.exec.mode: "deny"` y `"allowlist"` bloquean por completo la ejecución local del servidor de aplicaciones de Codex. Utilice `tools.exec.mode: "full"` únicamente cuando se desee intencionadamente una configuración sin aprobaciones.

Para obtener información sobre la configuración del servidor de aplicaciones, el orden de autenticación y el entorno de ejecución nativo de Codex, consulte [Entorno de ejecución de Codex](/es/plugins/codex-harness).

## Permisos del entorno de ejecución ACPX

Las sesiones ACPX no son interactivas, por lo que no pueden responder a una solicitud de permisos en una TTY. ACPX utiliza una configuración independiente en el nivel del entorno de ejecución bajo `plugins.entries.acpx.config`:

| Configuración                     | Valores          | Significado                                     |
| --------------------------------- | ---------------- | ----------------------------------------------- |
| `permissionMode`            | `approve-reads` | Aprueba automáticamente solo las lecturas.     |
| `permissionMode`            | `approve-all`   | Aprueba automáticamente las escrituras y los comandos del shell. |
| `permissionMode`            | `deny-all`      | Deniega todas las solicitudes de permisos.     |
| `nonInteractivePermissions` | `fail`          | Cancela la ejecución cuando se requeriría una solicitud. |
| `nonInteractivePermissions` | `deny`          | Deniega la solicitud y continúa cuando sea posible. |

Configure los permisos de ACPX por separado de las aprobaciones de ejecución de OpenClaw:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
openclaw gateway restart
```

Utilice `approve-all` como equivalente de emergencia en ACPX para una sesión del entorno de ejecución sin solicitudes. Para obtener información sobre la configuración y los modos de fallo, consulte [Configuración de agentes ACP](/es/tools/acp-agents-setup#permission-configuration).

## Elección de un modo

| Objetivo                                      | Configuración                                               |
| --------------------------------------------- | ----------------------------------------------------------- |
| Bloquear por completo los comandos en el host | `tools.exec.mode: "deny"`                                          |
| Permitir únicamente comandos cuya seguridad se conoce | `tools.exec.mode: "allowlist"`                                  |
| Consultar a una persona por cada nueva forma de comando | `tools.exec.mode: "ask"`                                  |
| Utilizar la revisión automática de Codex/OpenClaw antes de recurrir a personas | `tools.exec.mode: "auto"`                        |
| Omitir por completo las aprobaciones de ejecución en el host | `tools.exec.mode: "full"` más el archivo de aprobaciones del host correspondiente |
| Permitir que las sesiones ACPX no interactivas escriban y ejecuten | `plugins.entries.acpx.config.permissionMode: "approve-all"`                          |

Si un comando sigue solicitando aprobación o falla después de cambiar el modo, inspeccione ambas capas:

```bash
openclaw approvals get
openclaw exec-policy show
```

La ejecución en el host utiliza el resultado más estricto entre la configuración de OpenClaw y el archivo local de aprobaciones del host. Los permisos del entorno de ejecución ACPX no flexibilizan las aprobaciones de ejecución en el host, y las aprobaciones de ejecución en el host no flexibilizan las solicitudes del entorno de ejecución ACPX.

## Temas relacionados

- [Aprobaciones de ejecución](/es/tools/exec-approvals)
- [Aprobaciones de ejecución: opciones avanzadas](/es/tools/exec-approvals-advanced)
- [Entorno de ejecución de Codex](/es/plugins/codex-harness)
- [Configuración de agentes ACP](/es/tools/acp-agents-setup#permission-configuration)
