---
read_when:
    - Llamar a herramientas sin ejecutar un turno completo del agente
    - Creación de automatizaciones que requieren la aplicación de políticas de herramientas
summary: Invocar una sola herramienta directamente mediante el endpoint HTTP del Gateway
title: Las herramientas invocan la API
x-i18n:
    generated_at: "2026-07-26T05:42:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d07f765d63255e718d5e558b662589e77b2992538f43288cd83e6e3f2a06dda
    source_path: gateway/tools-invoke-http-api.md
    workflow: 16
---

El Gateway de OpenClaw expone un endpoint HTTP para invocar directamente una única herramienta. Siempre está habilitado y utiliza la autenticación del Gateway junto con la política de herramientas. Al igual que la superficie `/v1/*` compatible con OpenAI, la autenticación de portador con secreto compartido se considera acceso de operador de confianza para todo el Gateway.

- `POST /tools/invoke`
- Mismo puerto que el Gateway (multiplexación de WS + HTTP): `http://<gateway-host>:<port>/tools/invoke`
- Tamaño máximo predeterminado del cuerpo de la solicitud: 2 MB

## Autenticación

Utiliza la configuración de autenticación del Gateway.

Rutas habituales de autenticación HTTP:

- autenticación mediante secreto compartido (`gateway.auth.mode="token"` o `"password"`): `Authorization: Bearer <token-or-password>`
- autenticación HTTP de confianza con identidad (`gateway.auth.mode="trusted-proxy"`): enrute mediante el proxy configurado con reconocimiento de identidad y permita que este inyecte los encabezados de identidad necesarios
- autenticación abierta de entrada privada (`gateway.auth.mode="none"`): no se requiere ningún encabezado de autenticación

Notas:

- `mode="token"` utiliza `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
- `mode="password"` utiliza `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
- `mode="trusted-proxy"` requiere que la solicitud HTTP proceda de un origen de proxy de confianza configurado; los proxies de bucle invertido del mismo host requieren `gateway.auth.trustedProxy.allowLoopback = true` explícito.
- Los llamadores internos del mismo host que omiten el proxy pueden utilizar `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` como alternativa directa local. Cualquier evidencia de los encabezados `Forwarded`, `X-Forwarded-*` o `X-Real-IP` mantiene la solicitud en la ruta del proxy de confianza.
- Si se configura `gateway.auth.rateLimit` y se producen demasiados fallos de autenticación, el endpoint devuelve `429` con `Retry-After`.

## Límite de seguridad (importante)

Trate este endpoint como una superficie de **acceso completo de operador** para la instancia del Gateway.

- Aquí, la autenticación de portador HTTP no es un modelo de ámbitos restringidos por usuario.
- Un token o una contraseña válidos del Gateway para este endpoint deben tratarse como una credencial de propietario u operador.
- En los modos de autenticación mediante secreto compartido (`token` y `password`), el endpoint restablece los valores predeterminados normales de operador completo incluso si el llamador envía un encabezado `x-openclaw-scopes` más restringido.
- La autenticación mediante secreto compartido también trata las invocaciones directas de herramientas en este endpoint como turnos enviados por el propietario.
- Los modos HTTP de confianza con identidad (autenticación mediante proxy de confianza o `gateway.auth.mode="none"` en una entrada privada) respetan `x-openclaw-scopes` cuando está presente y, de lo contrario, recurren al conjunto normal de ámbitos predeterminados del operador.
- Mantenga este endpoint únicamente en el bucle invertido, la red de Tailscale o una entrada privada; no lo exponga directamente a la Internet pública.

Matriz de autenticación:

| Modo de autenticación                                                                    | Comportamiento                                                                                                                                                                                                                                                                                                                                                   |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token` o `password` + `Authorization: Bearer ...`                                      | Demuestra la posesión del secreto compartido del operador del Gateway. Ignora `x-openclaw-scopes` más restringidos. Restablece el conjunto completo de ámbitos predeterminados del operador: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Trata las invocaciones directas de herramientas como turnos enviados por el propietario. |
| HTTP de confianza con identidad (autenticación mediante proxy de confianza o `mode="none"` en una entrada privada) | Autentica una identidad externa de confianza o un límite de implementación. Respeta `x-openclaw-scopes` cuando está presente. Recurre al conjunto normal de ámbitos predeterminados del operador cuando el encabezado está ausente. Solo pierde la semántica de propietario cuando el llamador restringe explícitamente los ámbitos y omite `operator.admin`.                               |

## Cuerpo de la solicitud

```json
{
  "tool": "sessions_list",
  "action": "json",
  "args": {},
  "sessionKey": "main",
  "dryRun": false
}
```

Campos:

- `tool` / `name` (cadena, obligatorio): nombre de la herramienta que se invocará. `name` tiene prioridad si se envían ambos.
- `action` (cadena, opcional): se combina con `args.action` si el esquema de la herramienta admite una propiedad `action` y `args` todavía no ha establecido ninguna.
- `args` (objeto, opcional): argumentos específicos de la herramienta.
- `sessionKey` (cadena, opcional): clave de la sesión de destino. Si se omite o es `"main"`, el Gateway utiliza la clave de sesión principal configurada (respeta `session.mainKey` y el agente predeterminado, o `global` en el ámbito de sesión global).
- `agentId` (cadena, opcional): resuelve la clave de sesión de ese agente. Produce un error con `400` si entra en conflicto con un `sessionKey` explícito que ya está asignado a otro agente.
- `idempotencyKey` (cadena, opcional): se utiliza para derivar un identificador estable de llamada a herramienta para la invocación.
- `dryRun` (booleano, opcional): reservado para uso futuro; actualmente se ignora.

## Comportamiento de políticas y enrutamiento

La disponibilidad de las herramientas se filtra mediante la misma cadena de políticas que utilizan los agentes del Gateway:

- `tools.profile` / `tools.byProvider.profile`
- `tools.allow` / `tools.byProvider.allow`
- `agents.<id>.tools.allow` / `agents.<id>.tools.byProvider.allow`
- políticas de grupo (si la clave de sesión corresponde a un grupo o canal)
- política de subagentes (cuando se invoca con una clave de sesión de subagente)

Si la política no permite una herramienta, el endpoint devuelve **404**.

Notas importantes sobre los límites:

- Las aprobaciones de ejecución son mecanismos de protección para el operador, no un límite de autorización independiente para este endpoint HTTP. Si se puede acceder a una herramienta desde aquí mediante la autenticación del Gateway y la política de herramientas, `/tools/invoke` no añade una solicitud adicional de aprobación por llamada.
- Si se puede acceder a `exec` desde aquí, trátelo como una superficie de shell con capacidad de modificación. Denegar `write`, `edit`, `apply_patch` o las herramientas HTTP de escritura en el sistema de archivos no hace que la ejecución del shell sea de solo lectura.
- No comparta las credenciales de portador del Gateway con llamadores que no sean de confianza. Si necesita separar distintos límites de confianza, ejecute gateways independientes (preferiblemente con distintos usuarios o hosts del sistema operativo).

De forma predeterminada, el HTTP del Gateway también aplica una lista de denegación estricta (aunque la política de sesión permita la herramienta):

| Herramienta      | Motivo                                                       |
| ---------------- | ------------------------------------------------------------ |
| `exec`           | Ejecución directa de comandos (superficie de RCE)            |
| `spawn`          | Creación arbitraria de procesos secundarios (superficie de RCE) |
| `shell`          | Ejecución de comandos de shell (superficie de RCE)           |
| `fs_write`       | Modificación arbitraria de archivos en el host               |
| `fs_delete`      | Eliminación arbitraria de archivos en el host                |
| `fs_move`        | Movimiento o cambio de nombre arbitrario de archivos en el host |
| `apply_patch`    | La aplicación de parches puede reescribir archivos arbitrarios |
| `sessions_spawn` | Orquestación de sesiones; iniciar agentes remotamente constituye RCE |
| `sessions_send`  | Inyección de mensajes entre sesiones                         |
| `cron`           | Plano de control de automatización persistente               |
| `gateway`        | Plano de control del Gateway; impide la reconfiguración mediante HTTP |
| `nodes`          | La retransmisión de comandos de Node puede alcanzar `system.run` en hosts emparejados |

`cron`, `gateway` y `nodes` también son exclusivos del propietario: incluso fuera de esta lista de denegación predeterminada, los llamadores que no sean propietarios no pueden invocarlos en esta superficie.

Personalice la lista de denegación general mediante `gateway.tools`:

```json5
{
  gateway: {
    tools: {
      // Herramientas adicionales que se bloquearán mediante HTTP /tools/invoke
      deny: ["browser"],
      // Eliminar herramientas de la lista de denegación predeterminada para llamadores propietarios o administradores
      allow: ["gateway"],
    },
  },
}
```

`gateway.tools.allow` es una anulación de exposición, no una ampliación de ámbitos. En los modos HTTP con identidad, `cron`, `gateway` y `nodes` siguen sin estar disponibles para los llamadores sin identidad de propietario o administrador (`operator.admin`), incluso cuando aparecen en `gateway.tools.allow`. La autenticación de portador mediante secreto compartido sigue la regla de operador de plena confianza indicada anteriormente.

Para ayudar a que las políticas de grupo resuelvan el contexto, se pueden establecer opcionalmente:

- `x-openclaw-message-channel: <channel>` (ejemplo: `slack`, `telegram`)
- `x-openclaw-account-id: <accountId>` (cuando existen varias cuentas)
- `x-openclaw-message-to: <target>` (destino de entrega para la política de herramientas de mensajería)
- `x-openclaw-thread-id: <threadId>` (contexto del hilo para la política de herramientas de mensajería)

## Respuestas

| Estado | Significado                                                                                    |
| ------ | ---------------------------------------------------------------------------------------------- |
| `200`  | `{ ok: true, result }`                                                                         |
| `400`  | `{ ok: false, error: { type, message } }` (solicitud no válida o error en la entrada de la herramienta) |
| `401`  | No autorizado                                                                                  |
| `403`  | `{ ok: false, error: { type, message, requiresApproval? } }` (llamada a herramienta bloqueada por la política) |
| `404`  | Herramienta no disponible (no encontrada o no incluida en la lista de permitidas)              |
| `405`  | Método no permitido                                                                            |
| `408`  | Se agotó el tiempo de lectura del cuerpo de la solicitud                                        |
| `413`  | El cuerpo de la solicitud superó el tamaño máximo de la carga útil                              |
| `429`  | Autenticación limitada por frecuencia (`Retry-After` establecido)                          |
| `500`  | `{ ok: false, error: { type, message } }` (error inesperado de ejecución de la herramienta; mensaje depurado) |

## Ejemplo

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer secret' \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "sessions_list",
    "action": "json",
    "args": {}
  }'
```

## Contenido relacionado

- [Protocolo del Gateway](/es/gateway/protocol)
- [Herramientas y plugins](/es/tools)
