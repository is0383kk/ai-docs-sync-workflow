---
read_when:
    - Quieres que OpenClaw se ejecute las 24 horas del día, los 7 días de la semana, en un VPS en la nube (no en tu portátil)
    - Se busca un Gateway de nivel de producción y siempre activo en un VPS propio
    - Se desea tener control total sobre la persistencia, los binarios y el comportamiento de reinicio
    - Está ejecutando OpenClaw en Docker en Hetzner o un proveedor similar
summary: Ejecuta OpenClaw Gateway 24/7 en un VPS económico de Hetzner (Docker) con estado persistente y binarios preinstalados
title: Hetzner
x-i18n:
    generated_at: "2026-07-26T05:13:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ffebc0ce725fd219d13d0a556940327e70dab810b8fbee0b365c4870dc7109b
    source_path: install/hetzner.md
    workflow: 16
---

Ejecute un Gateway de OpenClaw persistente en un VPS de Hetzner mediante Docker, con estado duradero, binarios integrados en la imagen y un comportamiento de reinicio seguro.

Los precios de Hetzner cambian; elija el VPS Debian/Ubuntu más pequeño que se ajuste a sus necesidades y amplíelo si se producen errores por falta de memoria.

Se puede acceder al Gateway mediante el reenvío de puertos SSH desde su portátil o mediante la exposición directa del puerto si gestiona por su cuenta el cortafuegos y los tokens.

Recordatorio del modelo de seguridad:

- Los agentes compartidos por la empresa son adecuados cuando todos se encuentran dentro del mismo límite de confianza y el entorno de ejecución se usa exclusivamente para fines empresariales.
- Mantenga una separación estricta: VPS/entorno de ejecución dedicado y cuentas dedicadas; no use perfiles personales de Apple, Google, navegador ni gestor de contraseñas en ese host.
- Si los usuarios pueden actuar de forma hostil entre sí, sepárelos por gateway/host/usuario del sistema operativo.

Consulte [Seguridad](/es/gateway/security) y [Alojamiento en VPS](/es/vps).

Esta guía presupone que se usa Ubuntu o Debian en Hetzner. En otro VPS Linux, adapte los paquetes según corresponda. Para consultar el flujo genérico de Docker, consulte [Docker](/es/install/docker).

## Requisitos

- VPS de Hetzner con acceso root
- Acceso SSH desde su portátil
- Docker y Docker Compose
- Credenciales de autenticación del modelo
- Credenciales opcionales de proveedores (QR de WhatsApp, token de bot de Telegram, OAuth de Gmail)
- ~20 minutos

## Ruta rápida

1. Aprovisionar el VPS de Hetzner
2. Instalar Docker
3. Clonar el repositorio de OpenClaw
4. Crear directorios persistentes en el host
5. Configurar `.env` y `docker-compose.yml`
6. Integrar los binarios necesarios en la imagen
7. `docker compose up -d`
8. Verificar la persistencia y el acceso al Gateway

<Steps>
  <Step title="Aprovisionar el VPS">
    Cree un VPS con Ubuntu o Debian en Hetzner y conéctese como root:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    Trate el VPS como infraestructura con estado, no como infraestructura desechable.

  </Step>

  <Step title="Instalar Docker (en el VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    Verifique la instalación:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="Clonar el repositorio de OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    Esta guía compila una imagen personalizada para que los binarios integrados en ella sobrevivan a los reinicios.

  </Step>

  <Step title="Crear directorios persistentes en el host">
    Los contenedores de Docker son efímeros; todo el estado de larga duración debe residir en el host.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # Establezca como propietario al usuario del contenedor (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="Configurar las variables de entorno">
    Cree `.env` en la raíz del repositorio:

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Configure `OPENCLAW_GATEWAY_TOKEN` para gestionar el token estable del gateway mediante
    `.env`; de lo contrario, configure `gateway.auth.token` antes de depender de clientes
    entre reinicios. Si no se configura ninguno, OpenClaw utiliza un token exclusivo del entorno de ejecución para
    ese inicio. Genere una contraseña para el almacén de claves de `GOG_KEYRING_PASSWORD`:

    ```bash
    openssl rand -hex 32
    ```

    **No confirme este archivo en el repositorio.** Contiene variables de entorno del contenedor/entorno de ejecución, como
    `OPENCLAW_GATEWAY_TOKEN`. La autenticación almacenada mediante OAuth/claves de API de proveedores reside en el
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` montado.

  </Step>

  <Step title="Configuración de Docker Compose">
    Cree o actualice `docker-compose.yml`:

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # Recomendación: mantenga el Gateway limitado a la interfaz de bucle invertido en el VPS; acceda mediante un túnel SSH.
          # Para exponerlo públicamente, elimine el prefijo `127.0.0.1:` y configure el cortafuegos según corresponda.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` solo facilita el arranque inicial; no sustituye una configuración real del gateway. Configure de todos modos la autenticación (`gateway.auth.token` o contraseña) y un modo de enlace seguro para su despliegue.

  </Step>

  <Step title="Pasos compartidos del entorno de ejecución de la máquina virtual Docker">
    Siga la guía compartida del entorno de ejecución para el flujo común del host Docker:

    - [Integrar los binarios necesarios en la imagen](/es/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Compilar e iniciar](/es/install/docker-vm-runtime#build-and-launch)
    - [Qué persiste y dónde](/es/install/docker-vm-runtime#what-persists-where)
    - [Actualizaciones](/es/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Acceso específico de Hetzner">
    Después de realizar los pasos compartidos de compilación e inicio, abra el túnel.

    **Requisito previo:** asegúrese de que la configuración de sshd del VPS permita el reenvío TCP. Si
    reforzó la configuración de SSH, compruebe `/etc/ssh/sshd_config` y establezca:

    ```text
    AllowTcpForwarding local
    ```

    `local` permite los reenvíos locales de `ssh -L` desde su portátil, al tiempo que bloquea
    los reenvíos remotos desde el servidor. Si se establece en `no`, el túnel falla con:
    `channel 3: open failed: administratively prohibited: open failed`

    Después de confirmar que el reenvío TCP está habilitado, reinicie el servicio SSH
    (`systemctl restart ssh`) y ejecute el túnel desde su portátil:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    Abra `http://127.0.0.1:18789/` y pegue el secreto compartido configurado.
    Esta guía utiliza de forma predeterminada el token del gateway; si cambió a la autenticación mediante contraseña,
    use en su lugar la contraseña configurada.

  </Step>
</Steps>

El mapa de persistencia compartido se encuentra en [Entorno de ejecución de máquina virtual Docker](/es/install/docker-vm-runtime#what-persists-where).

## Infraestructura como código (Terraform)

Para los equipos que prefieren flujos de trabajo de infraestructura como código, una configuración de Terraform mantenida por la comunidad proporciona:

- Configuración modular de Terraform con gestión remota del estado
- Aprovisionamiento automatizado mediante cloud-init
- Scripts de despliegue (arranque inicial, despliegue, copia de seguridad/restauración)
- Refuerzo de seguridad (cortafuegos, UFW, acceso exclusivo mediante SSH)
- Configuración del túnel SSH para acceder al gateway

**Repositorios:**

- Infraestructura: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- Configuración de Docker: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

Este enfoque complementa la configuración de Docker anterior con despliegues reproducibles, infraestructura controlada por versiones y recuperación automatizada ante desastres.

<Note>
Mantenido por la comunidad. Para informar de problemas o realizar contribuciones, consulte los enlaces de los repositorios anteriores.
</Note>

## Pasos siguientes

- Configurar canales de mensajería: [Canales](/es/channels)
- Configurar el Gateway: [Configuración del Gateway](/es/gateway/configuration)
- Mantener OpenClaw actualizado: [Actualización](/es/install/updating)

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Fly.io](/es/install/fly)
- [Docker](/es/install/docker)
- [Alojamiento en VPS](/es/vps)
