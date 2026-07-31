---
read_when:
    - Implementación de OpenClaw en EasyRunner
    - Ejecución del Gateway detrás del proxy Caddy de EasyRunner
    - Elección de volúmenes persistentes y autenticación para un Gateway alojado
summary: Ejecuta el Gateway de OpenClaw en EasyRunner con Podman y Caddy
title: EasyRunner
x-i18n:
    generated_at: "2026-07-26T05:19:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cbde016a8bf7662d4b4a056a3d122a423264179daf70b5705e8f10b0dad5cb
    source_path: platforms/easyrunner.md
    workflow: 16
---

EasyRunner aloja el Gateway de OpenClaw como una pequeña aplicación en contenedores detrás de su
proxy Caddy. Esta guía presupone un host EasyRunner que ejecuta aplicaciones Compose compatibles
con Podman y termina HTTPS mediante Caddy.

## Antes de comenzar

- Un servidor EasyRunner con un dominio dirigido a él.
- La imagen oficial de OpenClaw (`ghcr.io/openclaw/openclaw`) o una compilación propia.
- Un volumen de configuración persistente para `/home/node/.openclaw`.
- Un volumen de espacio de trabajo persistente para `/home/node/.openclaw/workspace`.
- Un token o una contraseña robustos para el Gateway.

Mantenga habilitada la autenticación de dispositivos siempre que sea posible. Si el proxy inverso no puede transmitir
correctamente la identidad del dispositivo, corrija primero la configuración de proxies de confianza (consulte
[Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth)); utilice omisiones peligrosas de
la autenticación únicamente en una red totalmente privada y controlada por el operador.

## Aplicación Compose

Cree una aplicación EasyRunner con un archivo Compose con esta estructura:

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    restart: unless-stopped
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENCLAW_HOME: /home/node
      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace
    volumes:
      - openclaw-config:/home/node/.openclaw
      - openclaw-workspace:/home/node/.openclaw/workspace
    labels:
      caddy: openclaw.example.com
      caddy.reverse_proxy: "{{upstreams 1455}}"
    command: ["node", "openclaw.mjs", "gateway", "--bind", "lan", "--port", "1455"]

volumes:
  openclaw-config:
  openclaw-workspace:
```

Sustituya `openclaw.example.com` por el nombre de host del Gateway. Almacene
`OPENCLAW_GATEWAY_TOKEN` en el gestor de secretos o variables de entorno de EasyRunner en lugar de
incluirlo en la definición de la aplicación. De forma predeterminada, la imagen se vincula a la interfaz de bucle invertido,
por lo que se requiere el valor explícito `--bind lan --port 1455` en `command` para que Caddy pueda
acceder al contenedor.

## Configurar OpenClaw

Dentro del volumen de configuración persistente, mantenga el Gateway accesible únicamente a través
del proxy y exija autenticación:

```json5
{
  gateway: {
    bind: "lan",
    port: 1455,
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },
}
```

Si Caddy termina TLS para el Gateway, configure los ajustes de proxies de confianza para
la ruta exacta del proxy en lugar de deshabilitar globalmente las comprobaciones de autenticación. Consulte
[Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).

## Verificar

Desde la estación de trabajo:

```bash
openclaw gateway probe --url https://openclaw.example.com --token <token>
openclaw gateway status --url https://openclaw.example.com --token <token>
```

Desde el host EasyRunner, `GET /healthz` (actividad) y `GET /readyz`
(disponibilidad) no requieren autenticación y sustentan la comprobación de estado de contenedor
integrada en la imagen. Compruebe también en los registros de la aplicación que el Gateway esté escuchando y que no haya
errores de inicio relacionados con SecretRef, plugins ni autenticación de canales.

## Actualizaciones y copias de seguridad

- Descargue o compile la nueva imagen de OpenClaw y, después, vuelva a desplegar la aplicación EasyRunner.
- Haga una copia de seguridad del volumen `openclaw-config` antes de las actualizaciones. Contiene
  `openclaw.json`, `agents/<agentId>/agent/auth-profiles.json` y el estado de los paquetes
  de plugins instalados.
- Haga una copia de seguridad de `openclaw-workspace` si los agentes escriben allí datos persistentes de proyectos.
- Ejecute `openclaw doctor` después de actualizaciones importantes para detectar migraciones de configuración y
  advertencias del servicio.

## Solución de problemas

- `gateway probe` no puede conectarse: confirme que el nombre de host de Caddy apunta a la aplicación
  y que el contenedor escucha en `0.0.0.0:1455`.
- La autenticación falla: rote simultáneamente el token en los secretos de EasyRunner y en el comando
  del cliente local.
- Los archivos pertenecen al usuario raíz después de restaurarlos: la imagen se ejecuta como `node` (uid 1000);
  repare los volúmenes montados para que ese usuario pueda escribir en
  `/home/node/.openclaw` y `/home/node/.openclaw/workspace`.
- Los plugins del navegador o de canales fallan: compruebe si los binarios externos necesarios,
  la salida de red y las credenciales montadas están disponibles dentro del
  contenedor.
