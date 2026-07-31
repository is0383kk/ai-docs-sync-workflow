---
read_when:
    - Un cliente ve `rate limit exceeded for <method>`, `AUTH_RATE_LIMITED` o errores de bloqueo
    - Se desea ajustar `gateway.auth.rateLimit`
    - Está evaluando la protección contra ataques de fuerza bruta en un Gateway expuesto
    - Necesita saber qué superficies del Gateway están sujetas a limitación y cuáles son sus límites.
summary: 'Referencia de todos los límites de frecuencia del Gateway: bloqueos previos a la autenticación, limitación de solicitudes del navegador y los webhooks, mecanismo de protección de escritura del plano de control, límites de sesiones ACP y tiempo de espera entre reinicios'
title: Limitación de frecuencia
x-i18n:
    generated_at: "2026-07-26T04:41:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7aa37b65347610bedfb1db8f661e7ba75ef3cdfed0ba73c4ce53d80acace1e48
    source_path: gateway/security/rate-limiting.md
    workflow: 16
---

El Gateway aplica varios límites de frecuencia independientes. Protegen distintos
límites, se basan en distintas identidades y generan errores con formatos diferentes.
Esta página es la referencia para todos ellos.

Resumen:

| Superficie                          | Límite (predeterminado)           | Basado en                        | Configurable             |
| ----------------------------------- | --------------------------------- | -------------------------------- | ------------------------ |
| Fallos de autenticación (token/contraseña/dispositivo) | 10 fallos / 60 s, bloqueo de 5 min | IP + ámbito de credenciales      | `gateway.auth.rateLimit` |
| Fallos de autenticación WS con origen en navegador | igual, loopback **no** exento     | IP u origen de página desde loopback | `gateway.auth.rateLimit` |
| Fallos de autenticación de Webhook (`/hooks`) | 20 fallos / 60 s, bloqueo de 60 s | IP                               | no                       |
| RPC de escritura del plano de control | 30 solicitudes / 60 s por método | método + dispositivo + IP        | no                       |
| Creación de sesiones ACP            | 120 sesiones / 10 s               | instancia del traductor          | interno                  |
| Ciclos de reinicio del Gateway      | espera de 30 s entre reinicios    | proceso                          | no                       |

## Intentos de autenticación (antes de la autenticación)

Los intentos de autenticación fallidos se limitan por IP de cliente, antes de
procesar cualquier solicitud. Esta es la protección contra fuerza bruta para los Gateways expuestos.

- Solo cuentan las credenciales _incorrectas_. Las credenciales ausentes (un cliente que nunca
  envió un token) y las autenticaciones correctas no consumen el cupo; una
  autenticación correcta restablece el contador de esa IP.
- Valores predeterminados: 10 fallos cada 60 segundos y, después, un bloqueo de 5 minutos para esa IP.
- Loopback (`127.0.0.1` / `::1`) está exento de forma predeterminada para que las sesiones locales de la CLI
  no puedan quedar bloqueadas.
- Los contadores se delimitan por clase de credencial, por lo que una avalancha contra una superficie
  no desplaza a otra. Los ámbitos incluyen el token o la contraseña compartidos del Gateway,
  los tokens de dispositivo, el emparejamiento de Node, la reaprobación de Node emparejados,
  los tokens de arranque de dispositivos y la emisión de desafíos de watchOS.

Durante el bloqueo, los intentos de conexión generan el siguiente error:

```json
{
  "code": "INVALID_REQUEST",
  "message": "unauthorized: too many failed authentication attempts (retry later)",
  "retryable": true,
  "retryAfterMs": 297000,
  "details": {
    "code": "AUTH_RATE_LIMITED",
    "authReason": "rate_limited",
    "recommendedNextStep": "wait_then_retry"
  }
}
```

Los intentos desde otras IP (incluido loopback) no se ven afectados durante un bloqueo.

Se puede ajustar en `gateway.auth.rateLimit` dentro de `openclaw.json`:

```json
{
  "gateway": {
    "auth": {
      "rateLimit": {
        "maxAttempts": 10,
        "windowMs": 60000,
        "lockoutMs": 300000,
        "exemptLoopback": true
      }
    }
  }
}
```

Las entradas `AUTH_RATE_LIMITED` repetidas en el registro del Gateway indican que alguien está
intentando adivinar credenciales; consulte el [manual de exposición](/es/gateway/security/exposure-runbook).

### Conexiones con origen en navegador

Las conexiones WebSocket que incluyen una cabecera `Origin` del navegador usan los mismos
límites, pero con la exención de loopback **siempre desactivada**: una página maliciosa en
un navegador local sigue siendo un cliente no confiable, por lo que localhost no recibe ningún trato especial
en esa ruta. Cuando una conexión de este tipo llega _desde_ una dirección de loopback, sus
fallos se basan en el origen normalizado de la página (por ejemplo,
`browser-origin:https://evil.example`) en lugar de la IP de loopback compartida,
por lo que cada origen tiene su propio depósito; desde direcciones que no sean de loopback, la clave
sigue siendo la IP del cliente. Esto no es configurable.

### Webhooks

El punto de entrada HTTP `/hooks` tiene su propio limitador de fallos: 20
autenticaciones fallidas cada 60 segundos por IP de cliente y, después, un bloqueo de 60 segundos.
Loopback no está exento. Una autenticación correcta del hook restablece el contador. Las solicitudes
limitadas reciben una respuesta HTTP `429 Too Many Requests` sin formato con una cabecera `Retry-After`
(segundos). Los límites son fijos; si una integración legítima los alcanza,
corrija sus credenciales en lugar de reintentar con mayor intensidad.

## Escrituras del plano de control (protección posterior a la autenticación)

Los RPC administrativos de escritura (`config.apply`, `config.patch`, `plugins.install`,
`plugins.setEnabled`, `plugins.uninstall`, `update.run`, `worktrees.*`,
`gateway.restart.request`, ...) también se limitan por frecuencia **después** de
la autorización: 30 solicitudes cada 60 segundos, por método y por
`deviceId+clientIp`.

Esto no es un límite de seguridad —los invocadores ya poseen `operator.admin`—, sino
una protección que limita los bucles descontrolados de clientes o agentes que saturan operaciones
costosas. El uso interactivo nunca lo alcanza; cada método tiene su propio depósito, por lo que
activar o desactivar un plugin no consume el cupo de las escrituras de configuración.

Cuando se supera, la solicitud genera un error que permite reintentos:

```json
{
  "code": "UNAVAILABLE",
  "message": "rate limit exceeded for config.patch; retry after 35s",
  "retryable": true,
  "retryAfterMs": 34539,
  "details": { "method": "config.patch", "limit": "30 per 60s" }
}
```

Los clientes deben respetar `retryAfterMs`. El límite es fijo (no configurable);
los depósitos caducan por sí solos y el mantenimiento del Gateway los depura.

## Creación de sesiones ACP

El traductor ACP limita la creación de sesiones a 120 sesiones nuevas por cada intervalo de 10 segundos
y por instancia del traductor. Si se supera, la solicitud genera un error
cuyo mensaje incluye el tiempo de espera (no hay ningún campo estructurado `retryAfterMs`
en esta ruta):

```
Se superó el límite de frecuencia de creación de sesiones ACP para <method>; vuelva a intentarlo después de <n> s.
```

Esto limita a los clientes descontrolados que crean sesiones en un bucle; el uso normal del IDE y de
los agentes se mantiene muy por debajo de este límite.

## Espera entre reinicios

Las solicitudes de reinicio del Gateway se agrupan y, después, se aplica una espera de 30 segundos entre
ciclos de reinicio. Un reinicio solicitado durante el periodo de espera se programa para después de que este
termine, en lugar de rechazarse. Esto es independiente del limitador del plano de control
anterior: `gateway.restart.request` consume una posición del cupo del plano de control _y_
el reinicio resultante respeta el periodo de espera.

## Notas operativas

- Todos los limitadores se almacenan en memoria y funcionan por proceso; varios Gateways no
  comparten estado. Sustituir el proceso del Gateway borra los contadores que le pertenecen
  (bloqueos de autenticación, limitación de Webhook y depósitos del plano de control). El
  periodo de espera entre reinicios sobrevive deliberadamente a los ciclos de reinicio dentro del proceso —eso es
  precisamente lo que limita— y solo se restablece con el proceso. El límite de sesiones ACP
  pertenece a su instancia del traductor y se restablece cuando se vuelve a crear esa instancia,
  no al reiniciar el Gateway.
- Los mapas de depósitos están acotados (límites estrictos de entradas más depuración periódica), por lo que
  las avalanchas de claves únicas no pueden aumentar la memoria sin límite.
- Cuando un cliente está detrás de un proxy inverso, la IP efectiva es la IP resuelta
  del cliente; consulte la [autenticación de proxy de confianza](/es/gateway/trusted-proxy-auth) para saber cómo
  se validan las cabeceras del proxy antes de que puedan influir en ella.
- La señalización de reintentos varía según la superficie: los limitadores de RPC del Gateway devuelven
  `retryable: true` junto con `retryAfterMs`, el punto de entrada de Webhook usa HTTP 429
  con una cabecera `Retry-After`, y ACP incluye la espera en el mensaje de error.
  En todos los casos, espere el tiempo indicado antes de volver a intentarlo, en lugar de hacerlo
  inmediatamente.
