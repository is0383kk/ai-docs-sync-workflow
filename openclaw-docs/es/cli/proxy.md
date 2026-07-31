---
read_when:
    - Es necesario validar el enrutamiento del proxy administrado por el operador antes de la implementación
    - Necesita capturar localmente el tráfico de transporte de OpenClaw para depurarlo
    - Se desea inspeccionar sesiones del proxy de depuración, blobs o consultas predefinidas integradas
summary: Referencia de la CLI para `openclaw proxy`, incluida la validación del proxy administrado por el operador y el inspector local de capturas del proxy de depuración
title: Proxy
x-i18n:
    generated_at: "2026-07-26T04:34:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

Valida el enrutamiento mediante un proxy administrado por el operador, o ejecuta el proxy de depuración explícito local e inspecciona el tráfico capturado.

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate` realiza comprobaciones preliminares de un proxy de reenvío administrado por el operador. Los demás son herramientas de depuración para investigar el nivel de transporte: iniciar un proxy local que capture tráfico, ejecutar un comando secundario a través de él, enumerar sesiones de captura, consultar patrones de tráfico, leer blobs capturados y purgar los datos de captura locales.

## Validación

Comprueba la URL efectiva del proxy administrado por el operador procedente de `--proxy-url`, de la configuración (`proxy.proxyUrl`) o de `OPENCLAW_PROXY_URL`, en ese orden de precedencia. Informa de un problema de configuración si no hay ningún proxy habilitado y configurado; pasa `--proxy-url` para realizar una comprobación preliminar puntual sin modificar la configuración.

Las URL de proxy administrado usan `http://` para un servicio de escucha de proxy de reenvío sin cifrar, o `https://` cuando OpenClaw debe abrir una conexión TLS con el propio punto de conexión del proxy antes de enviar solicitudes de proxy. Usa `--proxy-ca-file` para confiar en una CA privada para esa conexión TLS.

De forma predeterminada, ejecuta:

- una comprobación **permitida** con `https://example.com/` (se puede sustituir o ampliar con `--allowed-url`; admite repetición)
- una comprobación **denegada** con un señuelo temporal de bucle invertido (se puede sustituir con `--denied-url`; admite repetición)

Los destinos `--denied-url` personalizados adoptan una política de denegación ante errores: tanto las respuestas HTTP como los fallos de transporte ambiguos cuentan como fallos, a menos que se pueda verificar de forma independiente una señal de denegación específica del despliegue. El señuelo de bucle invertido integrado es el único destino en el que un error de transporte se considera una prueba de bloqueo.

Añade `--apns-reachable` para abrir también un túnel CONNECT HTTP/2 de APNs a través del proxy y confirmar que el entorno aislado de APNs responde. La sonda envía intencionadamente un token de proveedor no válido, por lo que una respuesta `403 InvalidProviderToken` de APNs cuenta como señal de accesibilidad satisfactoria (no como fallo).

### Opciones

| Indicador                | Efecto                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | imprime JSON legible por máquinas                                                                                  |
| `--proxy-url <url>`      | valida esta URL de proxy `http://`/`https://` en lugar de la configuración o el entorno                            |
| `--proxy-ca-file <path>` | confía en este archivo de CA PEM para verificar mediante TLS un punto de conexión de proxy HTTPS                    |
| `--allowed-url <url>`    | destino que se espera que funcione correctamente a través del proxy (admite repetición)                            |
| `--denied-url <url>`     | destino que se espera que el proxy bloquee (admite repetición)                                                     |
| `--apns-reachable`       | verifica también que se pueda acceder al entorno aislado de APNs mediante HTTP/2 a través del proxy                |
| `--apns-authority <url>` | autoridad de APNs que se va a sondear (valor predeterminado: `https://api.sandbox.push.apple.com`; producción: `https://api.push.apple.com`) |
| `--timeout-ms <ms>`      | tiempo de espera por solicitud                                                                                     |

Finaliza con el código 1 cuando fallan la configuración del proxy o las comprobaciones de destino.

Consulta [Proxy de red](/es/security/network-proxy) para obtener orientación sobre el despliegue y la semántica de denegación.

## Proxy de depuración

`start` inicia un proxy local que captura tráfico e imprime su URL, la ruta del certificado de CA y la ruta de la base de datos de capturas; se detiene con Ctrl+C. De forma predeterminada, se vincula a `127.0.0.1`, salvo que se establezca `--host`.

`run` inicia un proxy de depuración local y, después, ejecuta `<cmd...>` (tras `--`) con el entorno del proxy aplicado y en su propia sesión de captura.

El reenvío ascendente directo del proxy de depuración abre sockets ascendentes para realizar diagnósticos. Cuando el modo de proxy administrado de OpenClaw está activo, el reenvío directo de solicitudes de proxy y túneles CONNECT está deshabilitado de forma predeterminada; establece `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1` únicamente para diagnósticos locales aprobados.

`coverage` imprime un informe JSON (`summary` y un `entries` por transporte) que indica qué transportes se capturan, cuáles solo funcionan mediante proxy y cuáles no tienen cobertura.

`sessions` enumera las sesiones de captura recientes (`--limit`; valor predeterminado: 20).

`query --preset <name>` ejecuta una consulta integrada sobre el tráfico capturado, cuyo ámbito puede limitarse opcionalmente a `--session <id>`. Valores preestablecidos:

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>` imprime el contenido sin procesar del blob de una carga útil capturada.

`purge` elimina todos los metadatos y blobs del tráfico capturado. Las capturas son datos de depuración locales; púrgalas al terminar.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Proxy de red](/es/security/network-proxy)
- [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth)
