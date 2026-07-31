---
read_when:
    - Quieres un Gateway en contenedor con Podman en lugar de Docker
summary: Ejecutar OpenClaw en un contenedor Podman sin privilegios de root
title: Podman
x-i18n:
    generated_at: "2026-07-26T05:10:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2db1f2b0413d7b9e1b2007aaae2da9d07fa44a1b52901d4a6cbc6274e54567f1
    source_path: install/podman.md
    workflow: 16
---

Ejecuta el Gateway de OpenClaw en un contenedor Podman sin privilegios de root, administrado por el usuario actual sin privilegios de root.

El modelo:

- Podman ejecuta el contenedor del Gateway.
- La CLI `openclaw` del host es el plano de control.
- De forma predeterminada, el estado persistente reside en el host, en `~/.openclaw`.
- La administración cotidiana utiliza `openclaw --container <name> ...` en lugar de `sudo -u openclaw`, `podman exec` o un usuario de servicio independiente.

## Requisitos previos

- **Podman** en modo sin privilegios de root
- **CLI de OpenClaw** instalada en el host
- **Opcional:** `systemd --user` si se desea el inicio automático administrado por Quadlet
- **Opcional:** `sudo` solo si se desea `loginctl enable-linger "$(whoami)"` para la persistencia tras el arranque en un host sin interfaz gráfica

## Inicio rápido

<Steps>
  <Step title="Configuración inicial">
    Desde la raíz del repositorio, ejecuta `./scripts/podman/setup.sh`.

    Esto compila `openclaw:local` en el almacén de Podman sin privilegios de root (o descarga `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` si se han establecido), crea `~/.openclaw/openclaw.json` con `gateway.mode: "local"` si no existe y crea `~/.openclaw/.env` con un `OPENCLAW_GATEWAY_TOKEN` generado si no existe.

    Variables de entorno opcionales para la compilación:

    | Variable | Efecto |
    | --- | --- |
    | `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` | Utiliza una imagen existente o descargada en lugar de compilar `openclaw:local` |
    | `OPENCLAW_IMAGE_APT_PACKAGES` | Instala paquetes apt adicionales durante la compilación de la imagen (también acepta el valor heredado `OPENCLAW_DOCKER_APT_PACKAGES`) |
    | `OPENCLAW_IMAGE_PIP_PACKAGES` | Instala paquetes de Python adicionales durante la compilación de la imagen; fija las versiones y utiliza únicamente índices de paquetes de confianza |
    | `OPENCLAW_EXTENSIONS` | Compila y empaqueta los plugins seleccionados compatibles e instala sus dependencias de ejecución |
    | `OPENCLAW_INSTALL_BROWSER` | Preinstala Chromium y Xvfb para la automatización del navegador (establece el valor en `1`) |

    Para usar en su lugar una configuración administrada por Quadlet (solo Linux y servicios de usuario de systemd):

    ```bash
    ./scripts/podman/setup.sh --quadlet
    ```

    También se puede establecer `OPENCLAW_PODMAN_QUADLET=1`.

  </Step>

  <Step title="Iniciar el contenedor del Gateway">
    ```bash
    ./scripts/run-openclaw-podman.sh launch
    ```

    Inicia el contenedor con el uid/gid del usuario actual mediante `--userns=keep-id` y monta mediante enlace el estado de OpenClaw dentro del contenedor.

  </Step>

  <Step title="Ejecutar la incorporación dentro del contenedor">
    ```bash
    ./scripts/run-openclaw-podman.sh launch setup
    ```

    A continuación, abre `http://127.0.0.1:18789/` y utiliza el token de `~/.openclaw/.env`.

    Autenticación del modelo: utiliza la autenticación administrada por OpenClaw durante la configuración (claves de API de Anthropic o autenticación OAuth del navegador/mediante código de dispositivo de OpenAI Codex para OpenAI respaldado por Codex). El iniciador de Podman no monta los directorios de credenciales de la CLI del host, como `~/.claude` o `~/.codex`, en el contenedor de configuración ni en el del Gateway. Los inicios de sesión existentes de la CLI del host son únicamente vías prácticas para el mismo host; para instalaciones en contenedores, conserva la autenticación del proveedor en el estado montado `~/.openclaw` que administra la configuración.

  </Step>

  <Step title="Administrar el contenedor en ejecución desde la CLI del host">
    ```bash
    export OPENCLAW_CONTAINER=openclaw
    ```

    A continuación, los comandos normales de `openclaw` se ejecutan automáticamente dentro de ese contenedor:

    ```bash
    openclaw dashboard --no-open
    openclaw gateway status --deep   # incluye un análisis adicional del servicio
    openclaw doctor
    openclaw channels login
    ```

    En macOS, la máquina de Podman puede hacer que el navegador no parezca local para el Gateway. Si la interfaz de control informa de errores de autenticación del dispositivo después del inicio, sigue las indicaciones de Tailscale en [Podman y Tailscale](#podman-and-tailscale).

  </Step>
</Steps>

El iniciador manual solo lee una pequeña lista permitida de claves relacionadas con Podman desde `~/.openclaw/.env` y pasa variables de entorno de ejecución explícitas al contenedor; no entrega el archivo de entorno completo a Podman.

<a id="podman-and-tailscale"></a>

## Podman y Tailscale

Para acceder mediante HTTPS o desde un navegador remoto, sigue la documentación principal de Tailscale.

Notas específicas de Podman:

- Mantén el host de publicación de Podman en `127.0.0.1`.
- Da preferencia a `tailscale serve` administrado por el host frente a `openclaw gateway --tailscale serve`.
- En macOS, si el contexto de autenticación del dispositivo del navegador local no es fiable, utiliza el acceso mediante Tailscale en lugar de soluciones provisionales con túneles locales ad hoc.

Consulta [Tailscale](/es/gateway/tailscale) e [Interfaz de control](/es/web/control-ui).

## Systemd (Quadlet, opcional)

Si se ejecutó `./scripts/podman/setup.sh --quadlet`, la configuración instala un archivo de Quadlet en `~/.config/containers/systemd/openclaw.container`.

| Acción | Comando                                    |
| ------ | ------------------------------------------ |
| Iniciar  | `systemctl --user start openclaw.service`  |
| Detener   | `systemctl --user stop openclaw.service`   |
| Estado | `systemctl --user status openclaw.service` |
| Registros   | `journalctl --user -u openclaw.service -f` |

Después de editar el archivo de Quadlet:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw.service
```

Para conservar el servicio después del arranque en hosts SSH o sin interfaz gráfica, habilita la permanencia para el usuario actual:

```bash
sudo loginctl enable-linger "$(whoami)"
```

El servicio de Quadlet generado mantiene una configuración predeterminada fija y reforzada: puertos `127.0.0.1` publicados (Gateway `18789`, puente `18790`), `--bind lan` dentro del contenedor, espacio de nombres de usuario `keep-id`, `OPENCLAW_NO_RESPAWN=1`, `Restart=on-failure` y `TimeoutStartSec=300`. Lee `~/.openclaw/.env` como `EnvironmentFile` de ejecución para valores como `OPENCLAW_GATEWAY_TOKEN`, pero no utiliza la lista permitida de anulaciones específicas de Podman del iniciador manual. Para utilizar puertos de publicación personalizados, un host de publicación personalizado u otras opciones de ejecución del contenedor, utiliza en su lugar el iniciador manual o edita `~/.config/containers/systemd/openclaw.container` directamente y, después, recarga y reinicia el servicio.

## Configuración, entorno y almacenamiento

- **Directorio de configuración:** `~/.openclaw`
- **Directorio del espacio de trabajo:** `~/.openclaw/workspace`
- **Archivo del token:** `~/.openclaw/.env`
- **Asistente de inicio:** `./scripts/run-openclaw-podman.sh`

El script de inicio y Quadlet montan mediante enlace el estado del host dentro del contenedor: `OPENCLAW_CONFIG_DIR` -> `/home/node/.openclaw`, `OPENCLAW_WORKSPACE_DIR` -> `/home/node/.openclaw/workspace`. De forma predeterminada, estos son directorios del host, no un estado anónimo del contenedor, por lo que `openclaw.json`, los `auth-profiles.json` de cada agente, el estado del canal/proveedor, las sesiones y el espacio de trabajo sobreviven a la sustitución del contenedor. La configuración también inicializa `gateway.controlUi.allowedOrigins` para `127.0.0.1` y `localhost` en el puerto publicado del Gateway, de modo que el panel local funcione con el enlace que no es de bucle invertido del contenedor.

Variables de entorno útiles para el iniciador manual (consérvalas en `~/.openclaw/.env`; el iniciador lee ese archivo antes de determinar los valores predeterminados finales del contenedor y la imagen):

| Variable                                        | Valor predeterminado          | Efecto                                 |
| ------------------------------------------ | ---------------- | -------------------------------------- |
| `OPENCLAW_PODMAN_CONTAINER`                | `openclaw`       | Nombre del contenedor                         |
| `OPENCLAW_PODMAN_IMAGE` / `OPENCLAW_IMAGE` | `openclaw:local` | Imagen que se ejecutará                           |
| `OPENCLAW_PODMAN_GATEWAY_HOST_PORT`        | `18789`          | Puerto del host asignado al puerto `18789` del contenedor  |
| `OPENCLAW_PODMAN_BRIDGE_HOST_PORT`         | `18790`          | Puerto del host asignado al puerto `18790` del contenedor  |
| `OPENCLAW_PODMAN_PUBLISH_HOST`             | `127.0.0.1`      | Interfaz del host para los puertos publicados     |
| `OPENCLAW_GATEWAY_BIND`                    | `lan`            | Modo de enlace del Gateway dentro del contenedor |
| `OPENCLAW_PODMAN_USERNS`                   | `keep-id`        | `keep-id`, `auto` o `host`           |

Si se utiliza un valor no predeterminado para `OPENCLAW_CONFIG_DIR` o `OPENCLAW_WORKSPACE_DIR`, establece las mismas variables tanto para `./scripts/podman/setup.sh` como para los comandos `./scripts/run-openclaw-podman.sh launch` posteriores; el iniciador local del repositorio no conserva las anulaciones de rutas personalizadas entre sesiones del shell.

## Actualización de imágenes

Después de volver a compilar o descargar una imagen nueva, reinicia el contenedor o el servicio de Quadlet.
En el primer inicio de una versión nueva de OpenClaw, el Gateway ejecuta reparaciones seguras del estado y de los
plugins antes de indicar que está listo.

Si el Gateway se cierra en lugar de quedar listo, ejecuta una vez la misma imagen con
`openclaw doctor --fix` sobre el mismo estado y configuración montados y, después, reinicia el
Gateway normalmente:

```bash
OPENCLAW_CONFIG_DIR="${OPENCLAW_CONFIG_DIR:-$HOME/.openclaw}"
OPENCLAW_WORKSPACE_DIR="${OPENCLAW_WORKSPACE_DIR:-$OPENCLAW_CONFIG_DIR/workspace}"
OPENCLAW_PODMAN_IMAGE="${OPENCLAW_PODMAN_IMAGE:-${OPENCLAW_IMAGE:-openclaw:local}}"

podman run --rm -it \
  --userns=keep-id \
  --user "$(id -u):$(id -g)" \
  -e HOME=/home/node \
  -e NPM_CONFIG_CACHE=/home/node/.openclaw/.npm \
  -v "$OPENCLAW_CONFIG_DIR:/home/node/.openclaw:rw" \
  -v "$OPENCLAW_WORKSPACE_DIR:/home/node/.openclaw/workspace:rw" \
  "$OPENCLAW_PODMAN_IMAGE" \
  openclaw doctor --fix
```

En hosts con SELinux, añade `,Z` a ambos montajes mediante enlace si Podman bloquea el acceso al
estado montado.

## Comandos útiles

- **Registros del contenedor:** `podman logs -f openclaw`
- **Detener el contenedor:** `podman stop openclaw`
- **Eliminar el contenedor:** `podman rm -f openclaw`
- **Abrir la URL del panel desde la CLI del host:** `openclaw dashboard --no-open`
- **Estado y comprobación del estado mediante la CLI del host:** `openclaw gateway status --deep` (sondeo RPC y análisis adicional del servicio)

## Solución de problemas

- **Permiso denegado (EACCES) en la configuración o el espacio de trabajo:** De forma predeterminada, el contenedor se ejecuta con `--userns=keep-id` y `--user <your uid>:<your gid>`. Asegúrate de que las rutas de configuración y del espacio de trabajo del host pertenezcan al usuario actual.
- **Inicio del Gateway bloqueado (falta `gateway.mode=local`):** Asegúrate de que `~/.openclaw/openclaw.json` exista y establezca `gateway.mode="local"`. `scripts/podman/setup.sh` lo crea si no existe.
- **El contenedor se reinicia después de actualizar una imagen:** Ejecuta el comando único `openclaw doctor --fix` de [Actualización de imágenes](#upgrading-images) y vuelve a iniciar el Gateway.
- **Los comandos de la CLI del contenedor se dirigen al destino incorrecto:** Utiliza `openclaw --container <name> ...` explícitamente o exporta `OPENCLAW_CONTAINER=<name>` en el shell.
- **`openclaw update` falla con `--container`:** Es lo esperado. Vuelve a compilar o descarga la imagen y, después, reinicia el contenedor o el servicio de Quadlet.
- **El servicio de Quadlet no se inicia:** Ejecuta `systemctl --user daemon-reload` y, después, `systemctl --user start openclaw.service`. En sistemas sin interfaz gráfica, también puede ser necesario `sudo loginctl enable-linger "$(whoami)"`.
- **SELinux bloquea los montajes mediante enlace:** No modifiques el comportamiento predeterminado del montaje; el iniciador añade automáticamente `:Z` en Linux cuando SELinux está en modo obligatorio o permisivo.

## Contenido relacionado

- [Docker](/es/install/docker)
- [Proceso en segundo plano del Gateway](/es/gateway/background-process)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
