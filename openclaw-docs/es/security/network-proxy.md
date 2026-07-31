---
read_when:
    - Se busca una defensa en profundidad contra ataques SSRF y de revinculación de DNS
    - Configuración de un proxy de reenvío externo para el tráfico del entorno de ejecución de OpenClaw
summary: Cómo enrutar el tráfico HTTP y WebSocket del entorno de ejecución de OpenClaw mediante un proxy de filtrado gestionado por el operador
title: Proxy de red
x-i18n:
    generated_at: "2026-07-26T05:22:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw puede enrutar el tráfico HTTP y WebSocket en tiempo de ejecución mediante un proxy directo gestionado por el operador. Esta es una defensa en profundidad opcional: control centralizado del tráfico saliente, mayor protección contra SSRF y auditabilidad de los destinos en el perímetro de la red. Dado que el proxy evalúa el destino al establecer la conexión, después de la resolución DNS e inmediatamente antes de abrir la conexión ascendente, también reduce el intervalo que aprovechan los ataques de revinculación de DNS entre una comprobación DNS anterior en el nivel de la aplicación y la conexión saliente real. Una única política de proxy también proporciona a los operadores un lugar centralizado donde aplicar reglas de destino, segmentación de red, límites de frecuencia o listas de destinos salientes permitidos sin tener que volver a compilar OpenClaw.

OpenClaw no incluye, descarga, inicia, configura ni certifica ningún proxy. Se ejecuta la tecnología de proxy que se ajuste al entorno; OpenClaw enruta a través de ella sus propios clientes HTTP y WebSocket.

## Configuración

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

También se puede establecer la URL mediante el entorno:

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` tiene prioridad sobre `OPENCLAW_PROXY_URL`. Una URL configurada activa el enrutamiento mediante el proxy gestionado; eliminar ambas URL lo desactiva.

| Clave                  | Tipo                                 | Valor predeterminado | Notas                                                                                                                                 |
| ---------------------- | ------------------------------------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | string                               | sin establecer       | URL del proxy directo `http://` o `https://`. Las credenciales insertadas en la URL se consideran confidenciales y se censuran en las instantáneas y los registros. |
| `proxy.tls.caFile`     | string                               | sin establecer       | Paquete de CA para verificar un punto de conexión de proxy `https://` firmado por una CA privada.                            |
| `proxy.loopbackMode`     | `gateway-only` \| `proxy` \| `block` | `gateway-only` | Controla el comportamiento de omisión del bucle invertido; véase más adelante.                                                       |

Para servicios de Gateway gestionados, almacene la URL en la configuración para que se conserve después de una reinstalación, en lugar de depender del entorno del proceso en primer plano:

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

La alternativa mediante la variable de entorno `OPENCLAW_PROXY_URL` resulta más adecuada para las ejecuciones en primer plano. Para utilizarla con un servicio instalado, añádala al entorno persistente del servicio (`$OPENCLAW_STATE_DIR/.env`, de forma predeterminada `~/.openclaw/.env`) y, a continuación, reinstale el servicio para que launchd/systemd/Scheduled Tasks la incorpore.

### Punto de conexión de proxy HTTPS con una CA privada

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile` verifica el certificado TLS propio del punto de conexión del proxy. No es una configuración de confianza MITM para los destinos, un certificado de cliente ni un sustituto de la política de destinos del proxy. Utilice `NODE_EXTRA_CA_CERTS` en su lugar únicamente cuando todo el proceso de Node deba confiar desde el inicio en una CA adicional (por ejemplo, un sistema empresarial de inspección TLS que vuelva a firmar todos los certificados de destinos HTTPS). Esa variable se aplica a todo el proceso y debe configurarse antes de iniciar Node, por lo que OpenClaw no puede aplicarla durante la ejecución como hace con `proxy.tls.caFile`. Para confiar en el punto de conexión del proxy HTTPS, se recomienda `proxy.tls.caFile`: su ámbito se limita al enrutamiento mediante el proxy gestionado, en lugar de abarcar todo el proceso.

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## Funcionamiento del enrutamiento

Con una URL de proxy válida, los procesos protegidos en tiempo de ejecución (`openclaw gateway run`, `openclaw node run`, `openclaw agent --local`) enrutan el tráfico saliente HTTP y WebSocket normal mediante el proxy:

```text
Proceso de OpenClaw
  clientes fetch, node:http, node:https y WebSocket  -> proxy del operador -> destino
```

Internamente, OpenClaw instala [Proxyline](https://github.com/openclaw/proxyline) como entorno de ejecución de enrutamiento en el nivel del proceso. Abarca `fetch`, los clientes basados en undici, `node:http`/`node:https`, los clientes WebSocket habituales y los túneles `CONNECT` creados por funciones auxiliares. Además, sustituye los agentes HTTP de Node proporcionados por el código que realiza la llamada para impedir que los agentes explícitos (incluidos `axios`, `got`, `node-fetch` y otros clientes similares basados en agentes de Node) omitan silenciosamente el proxy.

El esquema de la URL del proxy describe el salto de OpenClaw al proxy, no al destino final:

- `http://proxy.example:3128` — TCP sin cifrar hacia el proxy; OpenClaw envía solicitudes de proxy HTTP, incluido `CONNECT` para destinos HTTPS.
- `https://proxy.example:8443` — OpenClaw abre una conexión TLS con el propio proxy (y verifica el certificado del proxy) y, a continuación, envía solicitudes de proxy HTTP dentro de esa sesión.

El TLS del destino es independiente del TLS del punto de conexión del proxy: para un destino HTTPS, OpenClaw siempre solicita al proxy un túnel `CONNECT` e inicia el TLS del destino a través de dicho túnel.

Mientras el proxy está activo, OpenClaw borra `no_proxy`/`NO_PROXY`. Estas listas de omisión se basan en destinos; si se mantuvieran `localhost` o `127.0.0.1` en ellas, los destinos SSRF podrían omitir el proxy por completo. Al cerrarse, OpenClaw restaura el entorno de proxy anterior y restablece el estado de enrutamiento almacenado en caché.

Algunos plugins gestionan un transporte personalizado que requiere su propia configuración de proxy incluso cuando está activo el enrutamiento en el nivel del proceso. El cliente de la API de bots de Telegram utiliza su propio despachador HTTP/1 de undici y admite por separado el entorno de proxy del proceso, además de la alternativa `OPENCLAW_PROXY_URL`.

### Modo de bucle invertido del Gateway

Los clientes locales del plano de control del Gateway suelen conectarse a un WebSocket de bucle invertido, como `ws://127.0.0.1:18789`. `proxy.loopbackMode` controla si este tráfico omite el proxy gestionado:

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only, proxy, or block
```

Una configuración de `proxyUrl` o `OPENCLAW_PROXY_URL` activa el enrutamiento gestionado. Establezca
`proxy.enabled: false` únicamente como una opción avanzada de exclusión que conserva la URL almacenada
sin activarla.

| Modo                     | Comportamiento                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `gateway-only` (predeterminado) | OpenClaw registra la autoridad de bucle invertido activa del Gateway como excepción de conexión directa, por lo que el tráfico WebSocket local del Gateway se conecta sin utilizar el proxy. Se admiten puertos de bucle invertido personalizados porque la excepción se dirige al host y puerto configurados exactos. El plugin de navegador incluido registra el mismo tipo de excepción para las URL locales exactas de disponibilidad de CDP y WebSocket de DevTools de los navegadores gestionados iniciados por OpenClaw; el proveedor incluido de incrustaciones de memoria de Ollama dispone de una ruta directa protegida más restringida para el origen de incrustaciones de bucle invertido local del host configurado exacto. |
| `proxy`                  | No se registra ninguna excepción de bucle invertido; el tráfico de bucle invertido del Gateway y Ollama pasa por el proxy. Un proxy remoto debe poder enrutar el tráfico de vuelta al servicio de bucle invertido del host de OpenClaw (por ejemplo, mediante un nombre de host, una IP o un túnel accesibles). Un proxy remoto convencional resuelve `127.0.0.1`/`localhost` respecto a sí mismo, no respecto al host de OpenClaw.                                                                                                                                                                                        |
| `block`                  | OpenClaw deniega las conexiones del plano de control al bucle invertido del Gateway y las conexiones protegidas de incrustaciones al bucle invertido de Ollama antes de abrir un socket.                                                                                                                                                                                                                                                                                                                                                                                    |

La omisión del plano de control del Gateway se limita a `localhost` y a las URL con direcciones IP de bucle invertido literales: utilice `ws://127.0.0.1:18789`, `ws://[::1]:18789` o `ws://localhost:18789`. Los demás nombres de host se enrutan como tráfico normal.

### Contenedores

Para los comandos `openclaw --container ...`, OpenClaw reenvía `OPENCLAW_PROXY_URL` a la CLI secundaria destinada al contenedor cuando está configurada. La URL debe ser accesible desde el interior del contenedor: allí, `127.0.0.1` hace referencia al propio contenedor, no al host. OpenClaw rechaza las URL de proxy de bucle invertido para los comandos destinados a contenedores, a menos que se establezca `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` para omitir explícitamente esa comprobación.

## Términos relacionados con el proxy

- `proxy.enabled` / `proxy.proxyUrl` — enrutamiento saliente mediante un proxy directo para el tráfico saliente en tiempo de ejecución. Esta página.
- `gateway.auth.mode: "trusted-proxy"` — autenticación entrante mediante un proxy inverso con reconocimiento de identidad para acceder al Gateway. Consulte [Autenticación mediante un proxy de confianza](/es/gateway/trusted-proxy-auth).
- `openclaw proxy` — proxy local de depuración e inspector de capturas para desarrollo y asistencia. Consulte [openclaw proxy](/es/cli/proxy).
- `tools.web.fetch.useTrustedEnvProxy` — opción de inclusión para `web_fetch` que permite que un proxy HTTP(S) del entorno controlado por el operador resuelva el DNS, manteniendo de forma predeterminada la fijación estricta de DNS y la política de nombres de host. Consulte [Obtención web](/es/tools/web-fetch#trusted-env-proxy).
- Configuraciones de proxy específicas de un canal o proveedor — sustituciones específicas del propietario para un transporte concreto. Se recomienda el proxy de red gestionado para controlar de forma centralizada el tráfico saliente de todo el entorno de ejecución.

## Validación del proxy

La política de destinos del proxy constituye el verdadero perímetro de seguridad; OpenClaw no puede verificar que el proxy bloquee los destinos correctos. Configúrelo para:

- Vincularse únicamente al bucle invertido o a una interfaz privada de confianza, accesible solo para el proceso, host, contenedor o cuenta de servicio de OpenClaw.
- Resolver los destinos por sí mismo y bloquearlos por IP después de la resolución DNS, al establecer la conexión, tanto para HTTP sin cifrar como para los túneles HTTPS `CONNECT`.
- Rechazar las omisiones basadas en destinos para los intervalos de bucle invertido, privados, locales de enlace, de metadatos, multidifusión, reservados y de documentación.
- Evitar las listas de nombres de host permitidos a menos que se confíe plenamente en la ruta de resolución DNS.
- Registrar el destino, la decisión, el estado y el motivo, pero nunca los cuerpos de las solicitudes, los encabezados de autorización, las cookies ni otros datos confidenciales.
- Mantener la política bajo control de versiones y revisar los cambios como sensibles para la seguridad.

Realice la validación desde el mismo host, contenedor o cuenta de servicio que ejecuta OpenClaw:

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

Con un punto de conexión de proxy HTTPS que utilice una CA privada:

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| Indicador                | Propósito                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| `--proxy-url <url>`      | Validar esta URL en lugar de resolver la configuración o el entorno.  |
| `--proxy-ca-file <path>` | Paquete de CA para un endpoint de proxy HTTPS.                         |
| `--allowed-url <url>`    | Destino que se espera que funcione (repetible).                        |
| `--denied-url <url>`     | Destino que se espera que esté bloqueado (repetible).                  |
| `--apns-reachable`       | Verificar también que el proxy pueda tunelizar una sonda HTTP/2 directa de APNs del entorno aislado. |
| `--apns-authority <url>` | Sobrescribir la autoridad de APNs sondeada con `--apns-reachable`.     |
| `--timeout-ms <ms>`      | Tiempo de espera por solicitud.                                       |
| `--json`                 | Salida legible por máquina.                                           |

Si no hay disponible ningún valor de configuración, entorno o `--proxy-url`, el comando informa de un problema de configuración; se debe pasar `--proxy-url` para realizar una comprobación preliminar puntual antes de cambiar la configuración.

Sin `--allowed-url`/`--denied-url`, las comprobaciones predeterminadas son: `https://example.com/` debe funcionar y debe bloquearse un servidor señuelo temporal de bucle invertido al que el proxy no debe poder acceder. La comprobación de bucle invertido se supera si hay un fallo de transporte o una respuesta distinta de 2xx que no contiene el token por ejecución del señuelo; falla si se recibe una respuesta 2xx sin el token (un éxito inesperado procedente de algo distinto del señuelo) y, en especial, si alguna respuesta contiene el token coincidente, ya que esto demuestra que el proxy realmente reenvió un destino de bucle invertido que debería haber denegado. Los destinos `--denied-url` personalizados no disponen de dicho token señuelo, por lo que aplican un cierre seguro: cualquier respuesta HTTP cuenta como accesible (fallo), y un error de transporte se notifica como no concluyente, en lugar de considerarse un bloqueo demostrado, porque OpenClaw no puede confirmar si el proxy denegó un origen accesible o si se produjo algún otro error. `--apns-reachable` envía intencionadamente un token de proveedor no válido, por lo que una respuesta `403 InvalidProviderToken` cuenta como prueba de que el túnel llegó a Apple. El comando finaliza con `1` ante cualquier fallo de validación; las credenciales de la URL del proxy se ocultan tanto en la salida de texto como en la JSON.

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

Comprobación manual de `curl` (la solicitud pública debe funcionar; las solicitudes de bucle invertido y de metadatos deben ser bloqueadas por el propio proxy; `curl` por sí solo no puede distinguir la denegación de un proxy de un origen inaccesible como sí puede hacerlo el señuelo integrado de `openclaw proxy validate`):

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## Destinos cuyo bloqueo se recomienda

Lista de denegación inicial para cualquier proxy de reenvío, cortafuegos o política de salida. El clasificador SSRF propio de OpenClaw se encuentra en `src/infra/net/ssrf.ts` y `packages/net-policy/src/ip.ts` (`BLOCKED_HOSTNAMES`, `BLOCKED_IPV4_SPECIAL_USE_RANGES`, `BLOCKED_IPV6_SPECIAL_USE_RANGES`, el prefijo de pruebas comparativas RFC 2544 y el tratamiento de IPv4 incrustado para los formatos NAT64/6to4/Teredo/ISATAP/IPv4 mapeado); son referencias útiles, pero OpenClaw no exporta ni aplica estas reglas en el proxy externo.

| Intervalo o host                                                                      | Motivo para bloquear                                      |
| ------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| `127.0.0.0/8`, `localhost`, `localhost.localdomain`                                  | Bucle invertido IPv4                                      |
| `::1/128`                                                                            | Bucle invertido IPv6                                      |
| `0.0.0.0/8`, `::/128`                                                                | Direcciones no especificadas o de esta red                |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`                                      | Redes privadas RFC 1918                                   |
| `169.254.0.0/16`, `fe80::/10`                                                        | Enlace local, incluidas las rutas comunes de metadatos en la nube |
| `169.254.169.254`, `metadata.google.internal`                                        | Servicios de metadatos en la nube                         |
| `100.64.0.0/10`                                                                      | Espacio de direcciones compartido de NAT de nivel de operador |
| `198.18.0.0/15`, `2001:2::/48`                                                       | Intervalos de pruebas comparativas                        |
| `192.0.0.0/24`, `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, `2001:db8::/32` | Intervalos de uso especial y documentación                |
| `224.0.0.0/4`, `ff00::/8`                                                            | Multidifusión                                             |
| `240.0.0.0/4`                                                                        | IPv4 reservado                                            |
| `fc00::/7`, `fec0::/10`                                                              | Intervalos IPv6 locales o privados                        |
| `100::/64`, `2001:20::/28`                                                           | Intervalos IPv6 de descarte y ORCHIDv2                    |
| `64:ff9b::/96`, `64:ff9b:1::/48`                                                     | Prefijos NAT64 con IPv4 incrustado                        |
| `2002::/16`, `2001::/32`                                                             | 6to4 y Teredo con IPv4 incrustado                         |
| `::/96`, `::ffff:0:0/96`                                                             | IPv6 compatible con IPv4 e IPv6 con IPv4 mapeado          |

Añada cualquier otro host de metadatos o intervalo reservado que documente el proveedor de nube o la plataforma de red.

## Límites

| Superficie                                                   | Estado del proxy administrado                                                                                                                            |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`, `node:http`, `node:https`, clientes WebSocket comunes | Se enrutan mediante los hooks del proxy administrado cuando está configurado.                                                                             |
| HTTP/2 directo de APNs                                       | Se enruta mediante el asistente `CONNECT` administrado de APNs.                                                                                  |
| Bucle invertido del plano de control del Gateway             | Solo es directo para la URL local exacta de bucle invertido del Gateway configurada.                                                                      |
| Reenvío ascendente del proxy de depuración                   | Está desactivado mientras el modo de proxy administrado está activo, salvo que se habilite explícitamente para diagnósticos locales.                      |
| IRC                                                          | TCP/TLS sin procesar; el modo de proxy HTTP administrado no lo enruta mediante proxy. Establezca `channels.irc.enabled: false` si el despliegue exige que todo el tráfico de salida atraviese el proxy de reenvío. |
| Otras llamadas de cliente `net`, `tls` o `http2` sin procesar | Deben ser clasificadas por la protección de sockets sin procesar antes de integrarse.                                                                      |

- Esta cobertura corresponde al nivel de proceso para clientes HTTP/WebSocket de JavaScript, no a un entorno aislado de red a nivel del sistema operativo.
- Los sockets `net`, `tls` y `http2` sin procesar, los complementos nativos y los procesos secundarios ajenos a OpenClaw pueden eludir el enrutamiento de Node, salvo que hereden y respeten las variables de entorno del proxy. Las CLI secundarias bifurcadas de OpenClaw heredan la URL del proxy administrado y el estado de `proxy.loopbackMode`.
- Las WebUI locales del usuario y los servidores de modelos locales no están cubiertos por una omisión general de la red local; inclúyalos en la lista de permitidos de la política de proxy del operador si es necesario. La excepción es la ruta directa protegida del proveedor integrado de incrustaciones de memoria de Ollama, limitada al origen exacto de bucle invertido local del host obtenido de su `baseUrl` configurado; los hosts de Ollama de la LAN, la red de Tailscale, la red privada y la red pública siguen usando el proxy administrado.
- El reenvío ascendente directo del proxy de depuración local (para solicitudes de proxy y túneles `CONNECT`) está desactivado de forma predeterminada mientras el modo de proxy administrado está activo; habilítelo únicamente para diagnósticos locales aprobados.
- OpenClaw no inspecciona, prueba ni certifica la política del proxy. Considere los cambios en la política del proxy como cambios operativos sensibles para la seguridad.
