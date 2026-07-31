---
read_when:
    - Ejecutar o depurar el proceso del Gateway
summary: Guía operativa del servicio Gateway, su ciclo de vida y sus operaciones
title: Manual de operaciones del Gateway
x-i18n:
    generated_at: "2026-07-26T05:12:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

Use esta página para la puesta en marcha inicial y las operaciones posteriores del servicio Gateway.

<CardGroup cols={2}>
  <Card title="Solución de problemas avanzada" icon="siren" href="/es/gateway/troubleshooting">
    Diagnósticos basados en síntomas con secuencias exactas de comandos y firmas de registro.
  </Card>
  <Card title="Configuración" icon="sliders" href="/es/gateway/configuration">
    Guía de configuración orientada a tareas y referencia completa de configuración.
  </Card>
  <Card title="Gestión de secretos" icon="key-round" href="/es/gateway/secrets">
    Contrato de SecretRef, comportamiento de las instantáneas en tiempo de ejecución y operaciones de migración y recarga.
  </Card>
  <Card title="Contrato del plan de secretos" icon="shield-check" href="/es/gateway/secrets-plan-contract">
    Reglas exactas de destino/ruta de `secrets apply` y comportamiento de perfiles de autenticación que solo admiten referencias.
  </Card>
</CardGroup>

## Puesta en marcha local en 5 minutos

<Steps>
  <Step title="Iniciar el Gateway">

```bash
openclaw gateway --port 18789
# depuración/rastreo reflejados en stdio
openclaw gateway --port 18789 --verbose
# finalizar por la fuerza el proceso que escucha en el puerto seleccionado y, después, iniciar
openclaw gateway --force
```

  </Step>

  <Step title="Verificar el estado del servicio">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Referencia de estado correcto: `Runtime: running`, `Connectivity probe: ok` y una línea `Capability` que coincida con lo esperado. Use `openclaw gateway status --require-rpc` para demostrar el RPC con alcance de lectura, no solo la accesibilidad.

  </Step>

  <Step title="Validar la disponibilidad de los canales">

```bash
openclaw channels status --probe
```

Con un gateway accesible, esto ejecuta sondeos en vivo de los canales de cada cuenta y auditorías opcionales. Si el gateway no está accesible, la CLI recurre a resúmenes de canales basados únicamente en la configuración.

  </Step>
</Steps>

<Note>
La recarga de la configuración del Gateway supervisa la ruta del archivo de configuración activo (resuelta a partir de los valores predeterminados del perfil/estado, o `OPENCLAW_CONFIG_PATH` cuando se establece). El modo predeterminado es `gateway.reload.mode="hybrid"`. Después de la primera carga correcta, el proceso en ejecución utiliza la instantánea activa de la configuración en memoria; una recarga correcta sustituye esa instantánea de forma atómica.
</Note>

## Modelo de tiempo de ejecución

- Un proceso siempre activo para el enrutamiento, el plano de control y las conexiones de canales.
- Un único puerto multiplexado para:
  - Control/RPC mediante WebSocket
  - API HTTP (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - Rutas HTTP de Plugin, como la ruta opcional `/api/v1/admin/rpc`
  - Interfaz de control y enlaces
- Modo de enlace predeterminado: `loopback`. Dentro de un entorno de contenedor detectado, el valor predeterminado efectivo es `auto` (se resuelve como `0.0.0.0` para el reenvío de puertos), salvo que la publicación o el túnel de Tailscale estén activos, lo que siempre fuerza `loopback`.
- La autenticación es obligatoria de forma predeterminada. Las configuraciones con secreto compartido usan `gateway.auth.token` / `gateway.auth.password` (o `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`), y las configuraciones de proxy inverso que no usan bucle invertido pueden utilizar `gateway.auth.mode: "trusted-proxy"`.

## Endpoints compatibles con OpenAI

La superficie de compatibilidad de mayor impacto de OpenClaw:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Por qué este conjunto es importante:

- La mayoría de las integraciones de Open WebUI, LobeChat y LibreChat sondean primero `/v1/models`.
- Muchas canalizaciones de RAG y memoria esperan `/v1/embeddings`.
- Los clientes nativos para agentes prefieren cada vez más `/v1/responses`.

`/v1/models` prioriza los agentes: devuelve `openclaw`, `openclaw/default` y `openclaw/<agentId>` para cada agente configurado. `openclaw/default` es el alias estable que siempre se asigna al agente predeterminado configurado. Envíe `x-openclaw-model` cuando desee sustituir el proveedor/modelo del backend; de lo contrario, el modelo normal y la configuración de incrustaciones del agente seleccionado mantienen el control.

Todos estos se ejecutan en el puerto principal del Gateway y usan el mismo límite de autenticación del operador de confianza que el resto de la API HTTP del Gateway.

El RPC HTTP de administración (`POST /api/v1/admin/rpc`) es una ruta de Plugin independiente y desactivada de forma predeterminada para herramientas del host que no pueden usar RPC mediante WebSocket. Consulte [RPC HTTP de administración](/es/plugins/admin-http-rpc).

### Precedencia del puerto y el enlace

| Configuración       | Orden de resolución                                                     |
| ------------ | -------------------------------------------------------------------- |
| Puerto del Gateway | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| Modo de enlace    | CLI/sustitución → `gateway.bind` → `loopback` (o `auto` en contenedores) |

Los servicios del gateway instalados registran el valor resuelto de `--port` en los metadatos del supervisor. Después de cambiar `gateway.port`, ejecute `openclaw doctor --fix` o `openclaw gateway install --force` para que launchd/systemd/schtasks inicie el proceso en el puerto nuevo.

El inicio del Gateway usa el mismo puerto y enlace efectivos cuando genera los orígenes locales de la interfaz de control para enlaces que no son de bucle invertido. Por ejemplo, `--bind lan --port 3000` genera `http://localhost:3000` y `http://127.0.0.1:3000` antes de que se ejecute la validación en tiempo de ejecución. Añada explícitamente a `gateway.controlUi.allowedOrigins` cualquier origen de navegador remoto, como las URL de proxy HTTPS.

### Modos de recarga en caliente

| `gateway.reload.mode` | Comportamiento                                   |
| --------------------- | ------------------------------------------ |
| `off`                 | Sin recarga de la configuración                           |
| `hot`                 | Aplicar únicamente cambios seguros en caliente                |
| `restart`             | Reiniciar ante cambios que requieran recarga         |
| `hybrid` (predeterminado)    | Aplicar en caliente cuando sea seguro y reiniciar cuando sea necesario |

## Conjunto de comandos del operador

```bash
openclaw gateway status
openclaw gateway status --deep   # añade un análisis del servicio a nivel del sistema
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` sirve para detectar servicios adicionales (LaunchDaemons/unidades de sistema de systemd/schtasks), no para realizar un sondeo más profundo del estado de RPC.

## Varios gateways (mismo host)

La mayoría de las instalaciones deben ejecutar un gateway por máquina. Un solo gateway puede alojar varios agentes y canales. Solo se necesitan varios gateways cuando se busca intencionadamente el aislamiento o un bot de recuperación.

Comprobaciones útiles:

```bash
openclaw gateway status --deep
openclaw gateway probe
```

Qué cabe esperar:

- `gateway status --deep` puede informar de `Other gateway-like services detected (best effort)` y mostrar indicaciones de limpieza cuando aún existen instalaciones obsoletas de launchd/systemd/schtasks.
- `gateway probe` puede advertir sobre `multiple reachable gateway identities` cuando responden gateways distintos o cuando OpenClaw no puede demostrar que los destinos accesibles sean el mismo gateway. Un túnel SSH, una URL de proxy o una URL remota configurada que apunten al mismo gateway constituyen un único gateway con varios transportes, aunque los puertos de transporte sean diferentes.
- Si esto es intencionado, aísle los puertos, la configuración/estado y las raíces de los espacios de trabajo de cada gateway.

Lista de comprobación por instancia:

- `gateway.port` único
- `OPENCLAW_CONFIG_PATH` único
- `OPENCLAW_STATE_DIR` único
- `agents.defaults.workspace` único

Ejemplo:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

Configuración detallada: [/gateway/multiple-gateways](/es/gateway/multiple-gateways).

## Acceso remoto

Opción preferida: Tailscale/VPN.
Alternativa: túnel SSH.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Después, conecte localmente los clientes a `ws://127.0.0.1:18789`.

<Warning>
Los túneles SSH no eluden la autenticación del gateway. Para la autenticación mediante secreto compartido, los clientes deben seguir
enviando `token`/`password` incluso a través del túnel. Para los modos que incluyen identidad,
la solicitud también debe satisfacer esa ruta de autenticación.
</Warning>

Consulte: [Gateway remoto](/es/gateway/remote), [Autenticación](/es/gateway/authentication), [Tailscale](/es/gateway/tailscale).

## Supervisión y ciclo de vida del servicio

Use ejecuciones supervisadas para obtener una fiabilidad similar a la de producción.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Use `openclaw gateway restart` para los reinicios. No encadene `openclaw gateway stop` y `openclaw gateway start` como sustituto de un reinicio.

En macOS, `gateway stop` usa `launchctl bootout` de forma predeterminada. Esto elimina el LaunchAgent de la sesión de arranque actual sin conservar una desactivación, por lo que la recuperación automática de KeepAlive sigue funcionando después de fallos inesperados y `gateway start` lo vuelve a activar correctamente. Para impedir de forma persistente la reaparición automática tras los reinicios del sistema, pase `--disable`: `openclaw gateway stop --disable`.

Las etiquetas de LaunchAgent son `ai.openclaw.gateway` (predeterminada) o `ai.openclaw.<profile>` (perfil con nombre). `openclaw doctor` audita y corrige las desviaciones en la configuración del servicio.

  </Tab>

  <Tab title="Linux (usuario de systemd)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

Para mantener la persistencia después de cerrar sesión, active la permanencia:

```bash
sudo loginctl enable-linger $(whoami)
```

En un servidor sin interfaz gráfica ni sesión de escritorio, asegúrese también de que `XDG_RUNTIME_DIR` esté establecido (`export XDG_RUNTIME_DIR=/run/user/$(id -u)`) antes de volver a intentar los comandos `systemctl --user`.

Ejemplo de unidad de usuario manual cuando se necesita una ruta de instalación personalizada:

```ini
[Unit]
Description=Gateway de OpenClaw
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (nativo)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

El inicio administrado nativo de Windows usa una tarea programada denominada `OpenClaw Gateway`
(o `OpenClaw Gateway (<profile>)` para los perfiles con nombre). Si se deniega la creación de la tarea programada,
OpenClaw recurre a un iniciador por usuario en la carpeta Inicio
que apunta a `gateway.cmd` dentro del directorio de estado.

  </Tab>

  <Tab title="Linux (servicio del sistema)">

Use una unidad de sistema para hosts multiusuario/siempre activos.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

Use el mismo cuerpo de servicio que en la unidad de usuario, pero instálelo en
`/etc/systemd/system/openclaw-gateway[-<profile>].service` y ajuste
`ExecStart=` si el binario `openclaw` se encuentra en otra ubicación.

No permita también que `openclaw doctor --fix` instale un servicio de gateway a nivel de usuario para el mismo perfil/puerto. Doctor rechaza esa instalación automática cuando encuentra un servicio de gateway de OpenClaw a nivel del sistema; use `OPENCLAW_SERVICE_REPAIR_POLICY=external` cuando la unidad de sistema controle el ciclo de vida.

  </Tab>
</Tabs>

Los errores de configuración no válida terminan con el código `78`. Las unidades de systemd de Linux usan `RestartPreventExitStatus=78` para detener los nuevos intentos de inicio hasta que se corrija la configuración. launchd y el Programador de tareas de Windows no disponen de una regla equivalente para detenerse según el código de salida, por lo que el Gateway también conserva el historial de inicios rápidos fallidos e impide el inicio automático de las cuentas de canales/proveedores después de varios fallos de inicio. En ese modo seguro, el plano de control sigue iniciándose para permitir su inspección y reparación, las recargas en caliente de la configuración y `secrets.reload` rechazan los reinicios automáticos de los canales, y una solicitud explícita del operador mediante `channels.start` puede anular la restricción.

## Ruta rápida del perfil de desarrollo

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Los valores predeterminados incluyen estado/configuración aislados y el puerto base del gateway `19001`.

## Referencia rápida del protocolo (perspectiva del operador)

- La primera trama del cliente debe ser `connect`.
- El Gateway devuelve una trama `hello-ok` con un `snapshot` (`presence`, `health`, `stateVersion`, `uptimeMs`), además de los límites de `policy` (`maxPayload`, `maxBufferedBytes`, `tickIntervalMs`).
- `hello-ok.features.methods` / `events` son una lista de descubrimiento conservadora, no
  un volcado generado de todas las rutas auxiliares invocables.
- Solicitudes: `req(method, params)` → `res(ok/payload|error)`.
- Entre los eventos habituales se incluyen `connect.challenge`, `agent`, `chat`,
  `session.message`, `session.operation`, `session.tool`, el evento opcional
  `session.approval`, `sessions.changed`, `presence`, `tick`, `health`,
  `heartbeat`, eventos del ciclo de vida de vinculación/aprobación y `shutdown`.

Las ejecuciones del agente constan de dos etapas:

1. Confirmación inmediata de aceptación (`status:"accepted"`)
2. Respuesta final de finalización (`status:"ok"|"error"`), con eventos `agent` transmitidos entre ambas.

Consulte la documentación completa del protocolo: [Protocolo del Gateway](/es/gateway/protocol).

## Comprobaciones operativas

### Disponibilidad

- Abra una conexión WS y envíe `connect`.
- Se espera una respuesta `hello-ok` con una instantánea.

### Preparación

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Recuperación tras interrupciones

Los eventos no se reproducen de nuevo. Si hay interrupciones en la secuencia, actualice el estado (`health`, `system-presence`) antes de continuar.

## Indicadores habituales de error

| Indicador                                                      | Problema probable                                                                  |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | Enlace a una interfaz distinta de bucle invertido sin una ruta válida de autenticación del Gateway                           |
| `another gateway instance is already listening` / `EADDRINUSE` | Conflicto de puertos                                                                 |
| `Gateway start blocked: set gateway.mode=local`                | La configuración está establecida en modo remoto o falta `gateway.mode` en una configuración dañada |
| `unauthorized` durante la conexión                                  | Discrepancia de autenticación entre el cliente y el Gateway                                      |

Para consultar los procedimientos completos de diagnóstico, use [Solución de problemas del Gateway](/es/gateway/troubleshooting).

## Garantías de seguridad

- Los clientes del protocolo del Gateway fallan de inmediato cuando el Gateway no está disponible (sin respaldo implícito al canal directo).
- Las primeras tramas no válidas o que no sean de conexión se rechazan y se cierra la conexión.
- El apagado ordenado emite el evento `shutdown` antes de cerrar el socket.

## Contenido relacionado

- [Configuración](/es/gateway/configuration)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
- [Proceso en segundo plano](/es/gateway/background-process)
- [Estado](/es/gateway/health)
- [Doctor](/es/gateway/doctor)
- [Autenticación](/es/gateway/authentication)
- [Acceso remoto](/es/gateway/remote)
- [Gestión de secretos](/es/gateway/secrets)
