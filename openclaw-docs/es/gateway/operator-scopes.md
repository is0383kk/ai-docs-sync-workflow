---
read_when:
    - Depuración de errores por ámbitos de operador ausentes
    - Revisión de aprobaciones de vinculación de dispositivos o nodos
    - Añadir o clasificar métodos RPC del Gateway
summary: Roles de operador, ámbitos y comprobaciones en el momento de la aprobación para clientes del Gateway
title: Ámbitos del operador
x-i18n:
    generated_at: "2026-07-26T04:40:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

Los ámbitos de operador controlan lo que un cliente del Gateway puede hacer después de autenticarse.
Son una barrera de protección del plano de control dentro de un único dominio de operador de Gateway de confianza,
no un aislamiento hostil multiinquilino. Para lograr una separación sólida entre personas,
equipos o máquinas, ejecute Gateways separados con usuarios del SO o hosts distintos.

Relacionado: [Seguridad](/es/gateway/security), [Protocolo del Gateway](/es/gateway/protocol),
[Emparejamiento del Gateway](/es/gateway/pairing), [CLI de dispositivos](/es/cli/devices).

## Roles

Cada cliente WebSocket del Gateway se conecta con un rol:

- `operator`: clientes del plano de control, como la CLI, la interfaz de control, la automatización y
  los procesos auxiliares de confianza.
- `node`: hosts de capacidades (macOS, iOS, Android, sin interfaz gráfica) que exponen
  comandos mediante `node.invoke`.

Los métodos RPC de operador requieren el rol `operator`; los métodos originados por nodos
requieren el rol `node`.

## Niveles de ámbito

| Ámbito                  | Significado                                                                                                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | Estado de solo lectura, listas, catálogo, registros, lecturas de sesiones y otras llamadas que no realizan modificaciones.                                         |
| `operator.write`        | Acciones de operador que realizan modificaciones: enviar mensajes, invocar herramientas, actualizar la configuración de conversación/voz y retransmitir comandos de nodos. También satisface `operator.read`. |
| `operator.admin`        | Acceso administrativo. Satisface todos los ámbitos `operator.*`. Es obligatorio para modificar la configuración, realizar actualizaciones, usar hooks nativos, acceder a espacios de nombres reservados y efectuar aprobaciones de alto riesgo. |
| `operator.pairing`      | Gestión del emparejamiento de dispositivos y nodos: enumerar, aprobar, rechazar, eliminar, rotar y revocar.                                                          |
| `operator.approvals`    | API de aprobación de ejecución y plugins.                                                                                                                            |
| `operator.questions`    | Enumerar, leer, responder y resolver preguntas interactivas.                                                                                                         |
| `operator.talk.secrets` | Leer la configuración de conversación, incluidos los secretos.                                                                                                       |

Los ámbitos `operator.*` futuros y desconocidos requieren una coincidencia exacta, a menos que el autor de la llamada
ya posea `operator.admin`.

## El ámbito del método es solo la primera barrera

Cada RPC del Gateway tiene un ámbito de método con privilegios mínimos que decide si una
solicitud llega a su controlador. Los métodos que tienen en cuenta los parámetros derivan ese ámbito antes
del despacho, de modo que los errores de autorización tengan una única respuesta estructurada canónica:

- `agent` necesita `operator.write` para los turnos ordinarios y `operator.admin` para
  los comandos del ciclo de vida de la sesión `/new` o `/reset`.
- `node.invoke` necesita `operator.write` para los comandos de retransmisión ordinarios y
  `operator.admin` para `browser.proxy`, `fs.listDir` y `terminal.upload`.
- `talk.config` necesita `operator.read`; `includeSecrets: true` también necesita
  `operator.talk.secrets`.

A continuación, algunos controladores aplican comprobaciones más estrictas según el elemento concreto que se
aprueba o modifica:

- `device.pair.approve` es accesible con `operator.pairing`, pero al aprobar un
  dispositivo de operador solo se pueden emitir o conservar los ámbitos que ya posee el autor de la llamada.
- `node.pair.approve` es accesible con `operator.pairing` y después deriva ámbitos de
  aprobación adicionales de la lista de comandos declarada por el nodo pendiente.
- `chat.send` es un método con ámbito de escritura, pero los comandos de chat
  `/config set` y `/config unset` requieren además `operator.admin`,
  independientemente del ámbito de envío de chat del autor de la llamada.

Esto permite que los operadores con ámbitos inferiores realicen acciones de emparejamiento de bajo riesgo sin
hacer que todas las aprobaciones de emparejamiento estén reservadas exclusivamente a administradores.

Los RPC de modificación de sesiones se autorizan mediante los ámbitos de operador negociados,
independientemente del `client.id` o `client.mode` del cliente que se conecta. La identidad del
cliente puede seguir afectando a la política de conexión y autenticación de dispositivos, pero no
concede ni elimina la autoridad para modificar sesiones.

## Aprobaciones de emparejamiento de dispositivos

Los registros de emparejamiento de dispositivos son la fuente persistente de los roles y ámbitos aprobados.
Un dispositivo ya emparejado no obtiene silenciosamente un acceso más amplio: una reconexión
que solicita un rol o ámbitos más amplios crea una nueva solicitud de actualización pendiente.

Al aprobar una solicitud de dispositivo:

- Una solicitud sin rol de operador no necesita aprobación del ámbito de operador.
- Una solicitud para un rol de dispositivo que no sea de operador (por ejemplo, `node`) requiere
  `operator.admin`, aunque `device.pair.approve` solo necesite
  `operator.pairing`.
- Una solicitud de `operator.read`, `operator.write`, `operator.approvals`,
  `operator.questions`, `operator.pairing` o `operator.talk.secrets` requiere
  que el autor de la llamada ya posea ese ámbito o `operator.admin`.
- Una solicitud de `operator.admin` requiere `operator.admin`.
- Una solicitud de reparación sin ámbitos explícitos puede heredar los ámbitos del token de
  operador existente; si ese token tiene ámbito administrativo, la aprobación sigue requiriendo
  `operator.admin`.

Las sesiones no administrativas de secreto compartido y proxy de confianza solo pueden aprobar
solicitudes de dispositivos de operador dentro de sus propios ámbitos de operador declarados; la aprobación
de roles que no sean de operador está reservada exclusivamente a administradores, incluso cuando esas sesiones puedan usar
`operator.pairing` de otro modo.

En las sesiones con tokens de dispositivos emparejados, la gestión se limita al propio dispositivo, salvo que el autor de la llamada
tenga `operator.admin`: un autor de llamada que no sea administrador solo ve sus propias entradas de emparejamiento y
solo puede aprobar, rechazar, rotar, revocar o eliminar la entrada de su propio dispositivo.

## Aprobaciones de emparejamiento de nodos

Los métodos `node.pair.*` heredados usan un almacén de emparejamiento de nodos independiente propiedad del Gateway.
En cambio, los nodos WS usan el emparejamiento de dispositivos (`role: node`), pero se aplica el mismo
vocabulario de aprobación. Consulte [Emparejamiento del Gateway](/es/gateway/pairing) para saber cómo se relacionan ambos
almacenes.

`node.pair.approve` deriva ámbitos obligatorios adicionales de la lista de
comandos de la solicitud pendiente:

| Comandos declarados                                                                                                  | Ámbitos obligatorios                    |
| -------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| ninguno                                                                                                              | `operator.pairing`                      |
| comandos de nodo ordinarios                                                                                          | `operator.pairing` + `operator.write` |
| `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` o `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

Aprobar una declaración de nodo no habilita comandos que tengan una barrera independiente de lista de permitidos
en tiempo de ejecución. Por ejemplo, aprobar un nodo que declara
`computer.act` requiere emparejamiento y ámbito de escritura, pero solo registra la superficie.
Un administrador o propietario aún debe activar `computer.act`. Mientras permanezca
activado, invocarlo mediante `node.invoke` requiere ámbito de escritura, pero no ámbito
administrativo para cada acción.

El emparejamiento de nodos establece identidad y confianza; no sustituye la propia
política de aprobación de ejecución `system.run` de un nodo.

## Autenticación mediante secreto compartido

La autenticación mediante token o contraseña compartidos del Gateway se considera acceso de operador de confianza para
ese Gateway. Las superficies HTTP compatibles con OpenAI, `/tools/invoke` y los endpoints HTTP
del historial de sesiones restauran el conjunto predeterminado completo de ámbitos de operador para la
autenticación de portador mediante secreto compartido, incluso si el autor de la llamada envía ámbitos declarados más restringidos.

Los modos que incluyen identidad, como la autenticación mediante proxy de confianza o `none` de ingreso privado,
pueden seguir respetando los ámbitos declarados explícitamente. Use Gateways separados para una separación real de los
límites de confianza.
