---
read_when:
    - Ejecución o solución de problemas de configuraciones remotas del Gateway
summary: Acceso remoto mediante WS del Gateway, túneles SSH y tailnets
title: Acceso remoto
x-i18n:
    generated_at: "2026-07-26T04:41:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw ejecuta un Gateway (el principal) en un host y conecta todos los clientes a él. El Gateway gestiona las sesiones, los perfiles de autenticación, los canales y el estado; todo lo demás es un cliente.

- **Operadores** (usted o la aplicación para macOS): una conexión WebSocket directa por LAN/Tailnet es la opción más sencilla cuando se puede acceder al Gateway; el túnel SSH es la alternativa universal.
- **Nodos** (iOS/Android y otros dispositivos): se conectan al **WebSocket** del Gateway (mediante LAN/Tailnet o un túnel SSH).

## La idea principal

El WebSocket del Gateway se vincula a **loopback** de forma predeterminada, en el puerto `18789` (`gateway.port`). Para usarlo de forma remota, expóngalo mediante Tailscale Serve o una vinculación LAN/Tailnet de confianza, o reenvíe el puerto de loopback mediante SSH.

## Opciones de topología

| Configuración                             | Dónde se ejecuta el Gateway                                                                                    | Ideal para                                                                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Gateway siempre activo en su tailnet | Host persistente (VPS o servidor doméstico), al que se accede mediante Tailscale o SSH                                        | Portátiles que entran en suspensión con frecuencia, pero necesitan que el agente esté siempre activo. Consulte [exe.dev](/es/install/exe-dev) (máquina virtual sencilla) o [Hetzner](/es/install/hetzner) (VPS de producción). |
| Equipo de escritorio doméstico                      | Equipo de escritorio; el portátil se conecta de forma remota mediante el modo remoto de la aplicación para macOS (Settings → Connection → OpenClaw runs) | Mantener el agente en hardware que permanece encendido. Guía operativa: [acceso remoto en macOS](/es/platforms/mac/remote).                                       |
| Portátil                            | Portátil expuesto de forma segura mediante un túnel SSH o Tailscale Serve (mantenga `gateway.bind: "loopback"`)                | Configuraciones de una sola máquina. Consulte [Tailscale](/es/gateway/tailscale) y [Web](/es/web).                                                                       |

Para las configuraciones siempre activas y de portátil, se recomienda mantener `gateway.bind: "loopback"` y usar **Tailscale Serve** para la interfaz de control, o una vinculación LAN/Tailnet de confianza con `gateway.remote.transport: "direct"`. El túnel SSH es la alternativa que funciona desde cualquier máquina.

## Flujo de comandos (qué se ejecuta en cada lugar)

Un Gateway gestiona el estado y los canales; los nodos son periféricos. Ejemplo (mensaje de Telegram dirigido a una herramienta de nodo):

1. El mensaje de Telegram llega al **Gateway**.
2. El Gateway ejecuta el **agente**, que decide si debe llamar a una herramienta de nodo.
3. El Gateway llama al **nodo** mediante el WebSocket del Gateway (RPC `node.invoke`).
4. El nodo devuelve el resultado; el Gateway responde a Telegram.

Los nodos no ejecutan el servicio Gateway. Solo debe ejecutarse un Gateway por host, salvo que se ejecuten intencionadamente perfiles aislados (consulte [Varios gateways](/es/gateway/multiple-gateways)). El «modo nodo» de la aplicación para macOS es simplemente un cliente de nodo que utiliza el WebSocket del Gateway.

## Túnel SSH (CLI + herramientas)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Con el túnel activo, `openclaw health` y `openclaw status --deep` acceden al Gateway remoto mediante `ws://127.0.0.1:18789`. `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe` y `openclaw gateway call` también pueden dirigirse a una URL reenviada mediante `--url`.

<Note>
Sustituya `18789` por el valor configurado de `gateway.port` (o `--port` / `OPENCLAW_GATEWAY_PORT`).
</Note>

<Warning>
`--url` nunca recurre a las credenciales de configuración o del entorno. Proporcione explícitamente `--token` o `--password`; sin ellos, el cliente no envía credenciales y la conexión falla si el Gateway de destino requiere autenticación.
</Warning>

## Valores remotos predeterminados de la CLI

Guarde un destino remoto para que los comandos de la CLI lo utilicen de forma predeterminada:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

Cuando el Gateway solo utiliza loopback, mantenga la URL en `ws://127.0.0.1:18789` y abra primero el túnel SSH. En el transporte mediante túnel SSH de la aplicación para macOS, el nombre de host del Gateway detectado se introduce en `gateway.remote.sshTarget` (`user@host` o `user@host:port`); `gateway.remote.url` permanece como la URL local del túnel. Si el puerto remoto es distinto del local, establezca `gateway.remote.remotePort`.

La verificación de la clave del host es estricta de forma predeterminada (`gateway.remote.sshHostKeyPolicy: "strict"`). Establézcala en `"openssh"` para delegarla en la configuración efectiva de OpenSSH; revise la configuración SSH del usuario y del sistema antes de activarla.

Para un Gateway al que ya se puede acceder mediante una LAN o Tailnet de confianza, use el modo directo:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## Precedencia de credenciales

La resolución de credenciales del Gateway sigue un contrato compartido en las rutas de llamada, sondeo y estado, así como en la supervisión de aprobaciones de ejecución de Discord. El host del nodo utiliza el mismo contrato con una excepción para el modo local (ignora `gateway.remote.*`).

- Las credenciales explícitas (`--token`, `--password` o el `gatewayToken` de una herramienta) siempre tienen prioridad en las rutas de llamada que aceptan autenticación explícita.
- Seguridad de las sustituciones de URL:
  - El `--url` de la CLI nunca reutiliza credenciales implícitas de la configuración o del entorno.
  - El `OPENCLAW_GATEWAY_URL` del entorno solo puede usar credenciales del entorno (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`).
- Valores predeterminados del modo local:
  - token: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (se recurre al valor remoto solo cuando no se ha establecido el token local)
  - contraseña: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (se recurre al valor remoto solo cuando no se ha establecido la contraseña local)
- Valores predeterminados del modo remoto:
  - token: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - contraseña: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Excepción del modo local del host del nodo: se ignoran `gateway.remote.token` / `gateway.remote.password`.
- Las comprobaciones de token de sondeo y estado remotos son estrictas de forma predeterminada: solo usan `gateway.remote.token` (sin recurrir al token local) cuando se dirigen al modo remoto.
- Las sustituciones del entorno del Gateway solo usan `OPENCLAW_GATEWAY_*`.

## Acceso remoto a la interfaz de chat

WebChat no tiene un puerto HTTP independiente; la interfaz de chat de SwiftUI se conecta directamente al WebSocket del Gateway.

- Reenvíe `18789` mediante SSH (consulte la sección anterior) y conecte después los clientes a `ws://127.0.0.1:18789`.
- Para el modo directo por LAN/Tailnet, conecte los clientes a la URL privada `ws://` o segura `wss://` configurada.
- En macOS, el modo remoto de la aplicación gestiona automáticamente el transporte seleccionado.

## Modo remoto de la aplicación para macOS

La aplicación de la barra de menús de macOS gestiona la misma configuración de principio a fin: comprobaciones de estado remoto, WebChat y reenvío de Voice Wake. Guía operativa: [acceso remoto en macOS](/es/platforms/mac/remote).

## Reglas de seguridad (acceso remoto/VPN)

Mantenga el Gateway **solo en loopback**, salvo que tenga la certeza de que necesita una vinculación.

- **Loopback + SSH/Tailscale Serve** es la opción predeterminada más segura (sin exposición pública).
- Se acepta `ws://` sin cifrar para hosts de loopback, privados/LAN (RFC 1918), de enlace local, CGNAT, `.local` y `.ts.net`. Los hosts remotos públicos deben usar `wss://`.
- Las **vinculaciones que no sean de loopback** (`lan`/`tailnet`/`custom`, o `auto` cuando loopback no esté disponible) deben usar autenticación del Gateway: token, contraseña o un proxy inverso que tenga en cuenta la identidad con `gateway.auth.mode: "trusted-proxy"`.
- `gateway.remote.token` / `.password` son fuentes de credenciales del cliente; por sí solas no configuran la autenticación del servidor.
- Las rutas de llamada locales pueden recurrir a `gateway.remote.*` solo cuando `gateway.auth.*` no esté establecido.
- Si `gateway.auth.token` / `gateway.auth.password` se configura explícitamente mediante SecretRef y no se puede resolver, la resolución falla de forma cerrada (sin que el recurso al valor remoto oculte el fallo).
- `gateway.remote.tlsFingerprint` fija el certificado TLS remoto para `wss://`, incluido tanto el tráfico del operador/control como el nodo complementario en el modo directo de macOS. Si no hay una huella almacenada, macOS la fija en el primer uso solo después de superar la validación de confianza normal del sistema; los gateways autofirmados o con una CA privada necesitan una huella explícita o la opción de acceso remoto mediante SSH.
- **Tailscale Serve** puede autenticar el tráfico de la interfaz de control/WebSocket mediante encabezados de identidad cuando `gateway.auth.allowTailscale: true`. Los endpoints de la API HTTP no usan esa autenticación mediante encabezados y, en su lugar, siguen el modo de autenticación HTTP normal del Gateway. Este flujo sin token presupone que el host del Gateway es de confianza; establézcalo en `false` para usar autenticación mediante secreto compartido en todas partes.
- La autenticación mediante **proxy de confianza** espera de forma predeterminada un proxy que tenga en cuenta la identidad y no utilice loopback. Los proxies inversos de loopback en el mismo host requieren `gateway.auth.trustedProxy.allowLoopback = true` explícito.
- Trate el control desde el navegador como acceso de operador: solo mediante tailnet y con un emparejamiento de nodos deliberado.

Información detallada: [Seguridad](/es/gateway/security).

### macOS: túnel SSH persistente mediante LaunchAgent

Para clientes macOS, la configuración persistente más sencilla utiliza una entrada de configuración SSH `LocalForward` y un LaunchAgent que mantiene activo el túnel tras reinicios y fallos.

#### Paso 1: añadir la configuración SSH

Edite `~/.ssh/config`:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

Sustituya `<REMOTE_IP>` y `<REMOTE_USER>` por sus valores.

#### Paso 2: copiar la clave SSH (una sola vez)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### Paso 3: configurar el token del Gateway

```bash
openclaw config set gateway.remote.token "<your-token>"
```

Use `gateway.remote.password` en su lugar si el Gateway remoto utiliza autenticación mediante contraseña. `OPENCLAW_GATEWAY_TOKEN` sigue siendo válido como sustitución en el ámbito del shell, pero la configuración persistente del cliente remoto es `gateway.remote.token` / `gateway.remote.password`.

#### Paso 4: crear el LaunchAgent

Guárdelo como `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### Paso 5: cargar el LaunchAgent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

El túnel se inicia automáticamente al iniciar sesión, se reinicia si falla y mantiene activo el puerto reenviado.

<Note>
Si conserva un LaunchAgent `com.openclaw.ssh-tunnel` de una configuración anterior, descárguelo y elimínelo.
</Note>

#### Solución de problemas

```bash
# Comprobar si el túnel está en ejecución
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# Reiniciar el túnel
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# Detener el túnel
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| Entrada de configuración             | Qué hace                                                      |
| ------------------------------------ | ------------------------------------------------------------- |
| `LocalForward 18789 127.0.0.1:18789` | Reenvía el puerto local 18789 al puerto remoto 18789           |
| `ssh -N`                             | SSH sin ejecutar comandos remotos (solo reenvío de puertos)    |
| `KeepAlive`                          | Reinicia el túnel automáticamente si falla                     |
| `RunAtLoad`                          | Inicia el túnel cuando el LaunchAgent se carga al iniciar sesión |

## Relacionado

- [Tailscale](/es/gateway/tailscale)
- [Autenticación](/es/gateway/authentication)
- [Configuración del Gateway remoto](/es/gateway/remote-gateway-readme)
