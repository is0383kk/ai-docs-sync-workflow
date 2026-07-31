---
read_when:
    - Ejecución o depuración del proceso del Gateway
    - Investigación de la aplicación de instancia única
summary: 'Protección de instancia única del Gateway: bloqueo de archivo y enlace WebSocket/HTTP'
title: Bloqueo del Gateway
x-i18n:
    generated_at: "2026-07-26T04:39:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f5ac6d42c437b481c68a23a0aa4c00aeac9131acd76f3516ce3e949f325e265b
    source_path: gateway/gateway-lock.md
    workflow: 16
---

## Por qué

- Solo un proceso del Gateway debe controlar un directorio de estado; ejecute Gateways adicionales con perfiles, directorios de estado, configuraciones y puertos aislados.
- Permite superar fallos/SIGKILL sin dejar archivos de bloqueo obsoletos.
- Falla rápidamente con un error claro cuando otro Gateway ya controla el puerto.

## Tres capas

El inicio aplica el control de propiedad en tres pasos, en este orden:

1. El **bloqueo de propiedad del estado** adquiere un bloqueo asociado al directorio de estado canónico. Todos los Gateway participan, incluidos los Gateway iniciados con `OPENCLAW_ALLOW_MULTI_GATEWAY=1`, para que el mantenimiento destructivo de SQLite no entre en conflicto con un propietario activo.
2. El **bloqueo de configuración** adquiere el bloqueo histórico por configuración y registra el puerto de ejecución. El modo multi-Gateway omite esta instancia única de configuración, pero conserva el bloqueo de propiedad del estado.
3. La **vinculación del socket** vincula el listener HTTP/WebSocket (valor predeterminado: `ws://127.0.0.1:18789`) como un listener TCP exclusivo.

Cada capa puede fallar de forma independiente y genera su propio `GatewayLockError`.

### Bloqueos de estado y configuración

- La vigencia del bloqueo se determina mediante el PID registrado, la identidad de inicio del proceso de la plataforma cuando está disponible y la identidad del proceso del Gateway. Un propietario verificado conserva la autoridad durante el inicio antes de que su puerto comience a escuchar.
- Un coordinador de SQLite dedicado serializa la inspección de metadatos, la recuperación de propietarios obsoletos y la sustitución de bloqueos. Su transacción exclusiva se libera automáticamente si el proceso propietario falla.
- Si falta un archivo de bloqueo o el proceso propietario registrado ya no existe, el inicio recupera el bloqueo y continúa.
- Si cualquiera de los bloqueos está retenido activamente, el inicio vuelve a intentarlo durante un máximo de 5 segundos (valor predeterminado) antes de desistir:

  ```text
  GatewayLockError("el gateway ya está en ejecución (pid <pid>); tiempo de espera del bloqueo agotado después de <ms>ms")
  ```

### Vinculación del socket

- En `EADDRINUSE`, el inicio vuelve a intentar la vinculación hasta 20 veces en intervalos de 500ms (aproximadamente 10 segundos en total) para superar una ventana de `TIME_WAIT` tras la finalización reciente de un proceso.
- Si el puerto sigue en uso después de los reintentos:

  ```text
  GatewayLockError("otra instancia del gateway ya está escuchando en ws://127.0.0.1:<port>")
  ```

- Otros errores de vinculación:

  ```text
  GatewayLockError("no se pudo vincular el socket del gateway en ws://127.0.0.1:<port>: <cause>")
  ```

Al cerrarse, el Gateway cierra el servidor HTTP/WebSocket y elimina sus archivos
de bloqueo de estado y configuración.

## Notas operativas

- Si el puerto está ocupado por un proceso diferente que no es un Gateway, el error es el mismo; libere el puerto o elija otro con `openclaw gateway --port <port>`.
- `OPENCLAW_ALLOW_MULTI_GATEWAY=1` permite varias instancias de configuración/ejecución, no un estado mutable compartido. Cada instancia sigue necesitando un `OPENCLAW_STATE_DIR` único.
- Con un supervisor de servicios, un proceso nuevo del Gateway que encuentra cualquiera de los errores anteriores primero sondea `/healthz` en el proceso existente. Si ese proceso está en buen estado, el nuevo proceso permite que mantenga el control en lugar de fallar. En systemd, finaliza con el código `78`; el `RestartPreventExitStatus=78` de la unidad evita que `Restart=always` entre en un bucle debido a un bloqueo o un conflicto de `EADDRINUSE`. Si el proceso existente nunca alcanza un estado correcto, los reintentos del sondeo de estado tienen un límite temporal y, a continuación, el inicio falla con el error de bloqueo anterior en lugar de entrar en un bucle infinito.
- La aplicación para macOS mantiene su propia protección ligera mediante PID antes de iniciar el Gateway; el bloqueo de archivo y la vinculación del socket anteriores son los mecanismos reales de aplicación durante la ejecución.

## Contenido relacionado

- [Varios Gateway](/es/gateway/multiple-gateways) - ejecución de varias instancias con puertos únicos
- [Solución de problemas](/es/gateway/troubleshooting) - diagnóstico de `EADDRINUSE` y conflictos de puertos
