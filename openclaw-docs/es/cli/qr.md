---
read_when:
    - Quieres emparejar rápidamente una aplicación de nodo móvil con un Gateway
    - Necesita la salida del código de configuración para compartirla de forma remota/manual.
summary: Referencia de la CLI para `openclaw qr` (generar el código QR de vinculación móvil y el código de configuración)
title: QR
x-i18n:
    generated_at: "2026-07-26T05:03:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d60a58126eae7eec5979f28bb511a09fa52b68cdd73727fca0b2de74efa84a
    source_path: cli/qr.md
    workflow: 16
---

# `openclaw qr`

Genera un código QR de vinculación móvil y un código de configuración a partir de la configuración actual del Gateway.

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --limited
openclaw qr --url wss://gateway.example/ws
```

Las aplicaciones oficiales de OpenClaw para iOS y Android se conectan automáticamente cuando coinciden los metadatos de sus códigos de configuración. Si una solicitud sigue pendiente (por ejemplo, para un cliente no oficial o debido a metadatos que no coinciden), revísala y apruébala:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

## Opciones

- `--remote`: da preferencia a `gateway.remote.url`; recurre a `gateway.tailscale.mode=serve|funnel` si esa URL no está definida. Ignora `device-pair` del plugin `publicUrl`.
- `--url <url>`: reemplaza la URL del gateway utilizada en la carga útil
- `--public-url <url>`: reemplaza la URL pública utilizada en la carga útil
- `--token <token>`: reemplaza el token del gateway con el que se autentica el flujo de arranque
- `--password <password>`: reemplaza la contraseña del gateway con la que se autentica el flujo de arranque
- `--limited`: omite el acceso administrativo al Gateway del token de operador transferido
- `--setup-code-only`: imprime únicamente el código de configuración
- `--no-ascii`: omite la representación del código QR en ASCII
- `--json`: emite JSON (`setupCode`, `gatewayUrl`, `gatewayUrls` opcional, `auth`, `access`, `accessDowngraded` opcional, `urlSource`)

`--token` y `--password` son mutuamente excluyentes.

## Contenido del código de configuración

El código de configuración contiene un `bootstrapToken` opaco y de corta duración, no el token ni la contraseña compartidos del gateway. Para un punto de conexión `wss://` (o un bucle invertido en el mismo host), el flujo de arranque predeterminado emite:

- un token `node` principal con `scopes: []`
- un token de transferencia `operator` completo para dispositivos móviles nativos con `operator.admin`, `operator.approvals`, `operator.read`, `operator.talk.secrets` y `operator.write`

Utiliza `--limited` para conservar el mismo token de nodo y omitir `operator.admin` de la transferencia al operador. El ámbito de modificación de vinculaciones nunca se transfiere mediante un código de configuración.

La configuración de `ws://` en texto sin formato a través de la LAN sigue disponible, pero OpenClaw utiliza automáticamente el perfil limitado porque un observador de la red podría capturar el token de portador de arranque y adelantarse en su uso. Configura `wss://` o Tailscale Serve y, después, genera un código nuevo para obtener acceso completo.

## Resolución de la URL del Gateway

La vinculación móvil aplica un cierre seguro para las URL de gateway `ws://` de Tailscale o públicas: utiliza Tailscale Serve/Funnel o una URL de gateway `wss://` para ellas. Las direcciones LAN privadas y los hosts Bonjour `.local` siguen siendo compatibles mediante `ws://` sin cifrar, con acceso limitado del operador como se describe anteriormente.

Cuando la URL seleccionada del Gateway procede de `gateway.bind=lan`, OpenClaw también comprueba las rutas `tailscale serve status --json` persistentes. Cualquier raíz HTTPS de Serve que actúe como proxy del puerto de bucle invertido del Gateway activo se incluye como alternativa. El comando QR añade esta alternativa únicamente para `lan`; `custom` y `tailnet` conservan sus rutas anunciadas explícitamente. Los clientes actuales de iOS prueban las rutas anunciadas en orden y guardan la primera que sea accesible; el campo heredado `url` permanece sin cambios para los clientes antiguos.

Con `--remote`, se requiere `gateway.remote.url` o `gateway.tailscale.mode=serve|funnel`.

## Resolución de autenticación (sin `--remote`)

Cuando no se proporciona ninguna sustitución de autenticación mediante la CLI, las SecretRefs de autenticación del gateway local se resuelven de la siguiente manera:

| Condición                                                                                                                    | Se resuelve como                           |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| `gateway.auth.mode="token"`, o modo inferido sin una fuente de contraseña prevalente                                                  | `gateway.auth.token`                         |
| `gateway.auth.mode="password"`, o modo inferido sin un token prevalente de la autenticación o del entorno                                | `gateway.auth.password`                         |
| Tanto `gateway.auth.token` como `gateway.auth.password` están configurados (incluidas las SecretRefs) y `gateway.auth.mode` no está definido | falla; define `gateway.auth.mode` explícitamente |

## Resolución de autenticación (`--remote`)

Si las credenciales remotas efectivamente activas están configuradas como SecretRefs y no se proporciona ni `--token` ni `--password`, el comando las resuelve a partir de la instantánea del gateway activo. Si el gateway no está disponible, el comando falla de inmediato.

<Note>
Esta ruta de comando requiere un gateway compatible con el método RPC `secrets.resolve`. Los gateways antiguos devuelven un error de método desconocido.
</Note>

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Dispositivos](/es/cli/devices)
- [Vinculación](/es/cli/pairing)
